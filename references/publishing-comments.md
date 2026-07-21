# Publish Azure DevOps PR Comments

Use Azure DevOps Git REST API version 7.1 through the authenticated Azure DevOps CLI session. Keep payload files in a temporary directory outside the repository and do not print credentials.

Build every payload with a structured JSON serializer. Pass comment text, file paths, markers, and other dynamic values as data values; never interpolate them into shell commands, heredocs, or handwritten JSON. Validate the completed JSON file before submission, then send it only through the Azure DevOps CLI's file-input option such as `--in-file <payload-path>`. Do not place comment content directly on the command line.

## Read Before Writing

Use read operations to retrieve:

- fresh PR metadata, including repository ID, project, PR status, merge status, refs, and last merge source/target commit IDs
- all PR threads for duplicate-marker detection
- the latest PR iteration
- the latest iteration changes, including the exact `changeTrackingId` for each inline file
- effective PR policies when available, so the user knows whether active threads engage comment-resolution policy

Use `az devops invoke` with Git resources or the equivalent documented REST endpoints. Do not use reviewer-vote, PR-status, or PR-update resources.

## Comment Endpoint

Create threads only with:

```text
POST https://dev.azure.com/{organization}/{project}/_apis/git/repositories/{repositoryId}/pullRequests/{pullRequestId}/threads?api-version=7.1
```

Required scopes are `vso.threads_full` or the applicable code-write permission. If authorization fails, stop and report the missing permission; never request or expose a token in chat or command output.

## Idempotency Markers

Append an HTML comment to every published comment:

```text
<!-- codex-pr-review:{pr-id}:{head-sha}:{kind}:{identity} -->
```

Use `summary` as the summary identity. For a finding, derive a stable identity from severity, repository-relative path, start line, and title. Search all existing thread comments for the complete marker before posting. If found, treat that item as already published.

## General Summary

Use a closed thread so the informational summary does not create an unresolved-comment gate:

```json
{
  "comments": [
    {
      "parentCommentId": 0,
      "content": "<final review summary>\n\nReviewed commit: `<head-sha>`\n\n<!-- codex-pr-review:<pr-id>:<head-sha>:summary:summary -->",
      "commentType": 1
    }
  ],
  "status": 4
}
```

`status: 4` is `closed`. The verdict is plain comment text and must not be translated into a reviewer vote or PR status.

## Inline Finding

Use an active thread for an unresolved actionable finding:

```json
{
  "comments": [
    {
      "parentCommentId": 0,
      "content": "[P1] <title>\n\n<explanation>\n\nFailure scenario: <scenario>\n\nRecommended fix: <fix>\n\n<!-- codex-pr-review:<pr-id>:<head-sha>:finding:<identity> -->",
      "commentType": 1
    }
  ],
  "status": 1,
  "threadContext": {
    "filePath": "/<repository-relative-path>",
    "leftFileStart": null,
    "leftFileEnd": null,
    "rightFileStart": {
      "line": <start-line>,
      "offset": 1
    },
    "rightFileEnd": {
      "line": <end-line>,
      "offset": 1
    }
  },
  "pullRequestThreadContext": {
    "changeTrackingId": <exact-change-tracking-id>,
    "iterationContext": {
      "firstComparingIteration": <base-iteration-id>,
      "secondComparingIteration": <latest-iteration-id>
    }
  }
}
```

Line numbers start at 1. Resolve comparison iterations from Azure DevOps rather than assuming the first iteration. If the current right-side position or tracking ID cannot be proven, omit both context objects and create an active PR-level finding thread instead.

## Verification and Failure Handling

After each POST:

1. Require a successful response with a thread ID.
2. Confirm the response contains the expected marker and thread status.
3. Before the next POST, refresh PR metadata and rerun every publishing gate: PR status, merge status, clean working tree, exact source branch, source commit, and target commit.
4. If any gate changed, stop immediately and report the already-created thread IDs.

Never delete or edit a previously published thread automatically. A correction requires a new explicit user instruction naming the thread or requesting a new final review publication.
