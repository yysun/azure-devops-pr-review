---
name: azure-devops-pr-review
description: Review an Azure DevOps pull request by numeric PR ID from its checked-out source branch, deriving the exact target branch from Azure DevOps metadata and using its merge base with HEAD. Use when asked to inspect, audit, or code-review an Azure DevOps PR for actionable defects without changing the repository.
metadata:
  version: "1.0.1"
---

# Azure DevOps PR Review

Perform a read-only code review of one Azure DevOps pull request. Accept one positive numeric PR ID, treat the current `HEAD` as the PR source branch, and derive the target branch from that PR's Azure DevOps metadata.

## Preserve the Repository

- Do not modify, create, delete, or restore repository files.
- Do not use checkout, switch, pull, fetch, merge, rebase, reset, stash, clean, commit, format, autofix, snapshot-update, or code-generation commands.
- Do not post comments, votes, status changes, or other updates to Azure DevOps.
- Use only the existing local remote-tracking ref for the PR's exact target branch. If it is missing or may be stale, disclose that limitation instead of updating it.
- Capture `git status --short --branch` before reviewing. Preserve all existing user changes and exclude uncommitted changes from the PR diff.

## Establish the Review Context

1. Require a positive numeric Azure DevOps PR ID. Ask for it if absent or ambiguous.
2. Find the repository root with `git rev-parse --show-toplevel` and confirm that `HEAD` resolves. Read the repository-root `AGENTS.md` before proceeding when present.
3. Retrieve PR metadata with the read-only command `az repos pr show --id <PR_ID> --output json`. Record the title, description, `sourceRefName`, `targetRefName`, repository identity, and status.
4. Require `targetRefName` to have the form `refs/heads/<TARGET_BRANCH>`. Remove only the `refs/heads/` prefix, preserving the target branch's exact case and slashes. Resolve it locally as `refs/remotes/origin/<TARGET_BRANCH>`.
5. Never infer the target from the repository's default branch and never substitute `main` or `master`. For example, `refs/heads/QA` must resolve to `refs/remotes/origin/QA`.
6. Compare `sourceRefName` with the current branch. If they do not match, stop unless the user explicitly confirms that the current `HEAD` represents the PR source. Never change branches to resolve the mismatch.
7. If Azure DevOps metadata is unavailable, use an exact target branch explicitly supplied by the user and disclose that it could not be verified. If neither metadata nor an explicit target is available, stop and ask for the target rather than guessing.
8. Confirm that the derived target ref resolves to a commit:

```sh
git rev-parse --verify --end-of-options "refs/remotes/origin/<TARGET_BRANCH>^{commit}"
```

If it does not resolve, stop and report the missing ref. Do not fall back to another branch.

## Isolate the PR Changes

Compute the base exactly once:

```sh
git merge-base HEAD "refs/remotes/origin/<TARGET_BRANCH>"
```

Use the returned commit as `<BASE>`. Inspect only committed changes in `<BASE>..HEAD`:

```sh
git diff --name-status --find-renames <BASE> HEAD
git diff --stat <BASE> HEAD
git diff --find-renames --find-copies <BASE> HEAD --
```

After listing changed paths, read every nested `AGENTS.md` governing those paths and treat its conventions as review requirements.

Do not use the working-tree diff as PR content. Do not review unrelated commits on the target branch, pre-existing defects outside the changed behavior, or uncommitted local changes.

## Review the Implementation

Inspect the surrounding implementation, not only the patch:

- Open complete changed functions, classes, modules, schemas, migrations, configuration, and tests.
- Trace changed values through callers, callees, asynchronous boundaries, persistence layers, and public interfaces.
- Search for related call sites and repository conventions before concluding that behavior is wrong.
- Compare tests with each changed behavior and failure path.
- Use blame or history only when it resolves intent; do not report historical defects unless this PR makes them newly reachable or materially worse.

Review for:

- correctness and regressions
- security vulnerabilities and trust-boundary failures
- concurrency, cancellation, resource-lifetime, and error-handling problems
- material performance or scalability problems
- API, wire-format, schema, migration, and data compatibility
- missing or inadequate tests for behavior introduced by the PR
- violations of applicable `AGENTS.md` instructions

Run targeted tests or static analysis when they materially increase confidence and are read-only with respect to the repository. Prefer narrow commands with autofix, formatting, snapshot updates, coverage output, and generated artifacts disabled. Redirect caches or temporary output outside the repository when supported. If a useful check cannot be run without changing files, skip it and state why. Never install dependencies as part of the review.

## Decide Whether a Finding Is Actionable

Report a finding only when all of these are true:

- The PR introduced it or made it newly reachable.
- It has a concrete correctness, security, reliability, performance, compatibility, test, or repository-convention consequence.
- The failure scenario is realistic and can be explained precisely.
- The author can take a specific corrective action.

Do not report style preferences, generic hardening, praise, questions without a defect, speculative risks without a plausible trigger, or missing tests that do not cover a meaningful changed risk. Consolidate findings with the same root cause.

Assign the lowest severity justified by impact:

- **P0 — catastrophic or security-critical:** broad compromise, irreversible loss, or systemic outage with a credible trigger.
- **P1 — blocking correctness issue:** common or high-impact behavior is broken and the PR should not merge.
- **P2 — important but non-blocking:** a real defect with bounded impact or a material coverage gap around risky changed behavior.
- **P3 — minor improvement:** a small but concrete defect worth fixing; never use P3 for taste alone.

## Report the Review

Order findings by severity, then by file location. For every finding provide:

```markdown
### [P1] Concise defect title

- **Location:** `path/to/file.ext:line-start-line-end`
- **Explanation:** Why the changed implementation is wrong.
- **Failure scenario:** The exact input, state, or sequence that triggers the problem and its consequence.
- **Recommended fix:** A concrete correction, including the test that should protect it when relevant.
```

Use exact, tight line ranges from the `HEAD` version and anchor each finding to changed lines whenever possible. For a deletion-only defect, cite the nearest surviving changed line that exposes the regression and explain what was removed.

After the findings, always finish with these four sections:

1. **Overall verdict** — `Request changes`, `Approve with comments`, or `Approve`, with one sentence of rationale.
2. **PR summary** — summarize the behavior introduced by `<BASE>..HEAD`, not the PR description alone.
3. **Test coverage assessment** — list checks run and their outcomes, then identify meaningful untested changed behavior.
4. **Assumptions and unverified areas** — include unavailable metadata, the exact target ref used, stale or missing refs, environment limits, skipped checks, and source/target mismatches.

If there are no actionable findings, begin the report with **No actionable findings.** Do not invent a finding to fill the format.
