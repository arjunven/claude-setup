---
name: address-pr-feedback
description: Fetch PR review feedback (conversation + inline comments) and address it. Pass a PR number or URL, or omit to use the current branch's PR.
allowed-tools: Bash(gh repo view:*),Bash(gh pr view:*),Bash(gh api:*),Bash(gh pr diff:*),Bash(git add:*),Bash(git commit:*),Bash(git push:*)
---

# Skill: /address-pr-feedback [PR]

Fetch all review feedback on a PR and address it.

## Arguments

- `[PR]` — a PR number, URL, or omit to use the current branch's PR.

## Resolve the PR number

If no PR was given, run `gh pr view --json number --jq .number` to get the current branch's PR number.

## Resolve the repo

Run `gh repo view --json nameWithOwner --jq .nameWithOwner` to get the `{owner}/{repo}` for API calls.

## Fetching feedback

Run these in parallel:

- `gh pr view <number> --comments` — conversation-level comments
- `gh api repos/{owner}/{repo}/pulls/<number>/comments --paginate` — inline review comments (includes file path, line, and diff context)
- `gh pr diff <number>` — current diff for context when addressing inline comments

Include bot review comments (e.g. Claude auto-review) — these are often the first round of feedback to address.

## Addressing feedback

Collect **all** feedback — owner inline comments, conversation comments, and bot review suggestions (including non-blocking "nits" and "consider" items). Bot suggestions like refactoring repetitive code or extracting constants are cheap wins — fix them unless the user says otherwise.

Summarize all feedback items in a status table before starting work:

| # | Source | Item | Status |
|---|--------|------|--------|
| 1 | @user | Description of feedback | Fixed / Skipped (reason) / Flagged |

- **Source**: who left the comment (e.g. `@username`, `bot review`)
- **Status values**: `Fixed`, `Skipped — not blocking`, `Flagged — ambiguous`

Work through each item: read the relevant code, make the fix, move on. Flag anything ambiguous to the user instead of guessing. After all fixes, update the table with final statuses and present it to the user.

After all fixes, run `/lint`, commit, and push.

## Summary

After pushing, present the final status table to the user showing what was addressed, what was skipped and why, and any items flagged for their input.
