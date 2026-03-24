---
name: review-pr
description: Review a pull request for bugs, security issues, and rules violations
allowed-tools: mcp__github_inline_comment__create_inline_comment,Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh pr comment:*)
---

# Review PR

Review the pull request specified by REPO and PR_NUMBER.

## Step 1: Read project rules

Read `.claude/CLAUDE.md` and all `.claude/rules/*.md` files from the checkout to understand project-specific standards.

## Step 2: Fetch the diff

Run `gh pr diff $PR_NUMBER --repo $REPO` to get the full set of changes.

## Step 3: Review — high-signal only

Flag **only** definitive issues. Do not flag uncertain, subjective, or pre-existing problems.

**Flag these:**
- Bugs and logic errors that produce wrong results regardless of input
- Security: SQL/command injection, missing auth/permission checks, insecure direct object references, user input crossing trust boundaries without validation, sensitive data in logs or responses
- Performance: N+1 query patterns, blocking I/O in an async function, unclosed DB sessions or HTTP client connections, missing DB indexes on new FK columns
- Missing error handling at real failure points: unhandled DB errors, bare `except` swallowing exceptions, external API calls with no error path
- Workarounds masking root causes: per-call-site guards for a problem that has a framework/library-level solution (e.g. SQLAlchemy `JSON(none_as_null=True)` vs guarding every assignment), defensive checks that paper over an upstream bug instead of fixing it, repeated boilerplate that could be solved by configuration
- Missing tests for new non-trivial logic (new public functions, new API endpoints, new edge cases)
- Weak or missing cryptography: hardcoded secrets, MD5/SHA1 for passwords, missing HTTPS enforcement
- Race conditions in async code: shared mutable state across concurrent requests, time-of-check-time-of-use patterns
- Algorithm complexity red flags: O(n^2) or worse loops over unbounded data
- Missing test coverage for boundary conditions: empty inputs, zero values, maximum limits
- Clear violations of the `.claude/rules/` files read in Step 1 — quote the specific rule

**Do not flag:**
- Type errors — already caught by `ty` (Python) and `tsc` (TypeScript) in CI
- Style, formatting, naming — already caught by `ruff` and `biome` in CI
- Pre-existing issues in code not touched by this PR
- Issues that only occur under specific uncertain conditions
- Subjective quality suggestions or improvements

## Step 4: Validate

Before posting anything, re-read the relevant diff hunks for each finding. Confirm:
- The issue is in code **added or modified** by this PR (not pre-existing)
- It is a definitive problem, not a potential one
- You are not misreading the context

Discard any finding that does not pass this check.

## Step 5: Post feedback

- Use `mcp__github_inline_comment__create_inline_comment` for line-specific issues
- Post one comment per unique issue — no duplicates
- If there are findings, also post a brief top-level summary via `gh pr comment $PR_NUMBER --repo $REPO --body "..."`
- If there are no findings, post a top-level comment listing what was checked and confirming no issues, e.g. "Reviewed N files changed. Checked for bugs, security, performance, async patterns, and rules violations — no issues found."
