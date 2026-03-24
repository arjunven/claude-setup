---
name: send
description: Implement a task and ship it — commit, push, open a PR as ready for review, and address review feedback. Pass a Linear ticket (e.g. MER-381) or a description of the task. Works from any context (worktree, feature branch, or main).
allowed-tools: Bash(gh pr create:*),Bash(gh pr checks:*),Bash(gh pr view:*),Bash(gh api:*),Write(/tmp/pr-body-*)
---

# Skill: /send <task-or-ticket>

Implement a task, ship it as a PR ready for review, and address the first round of automated review feedback.

## Shell discipline (CRITICAL)

**NEVER compound `cd` with other commands** — `cd /path && command` is FORBIDDEN. Always `cd` in one Bash call, then run the next command in a separate Bash call. This applies to ALL commands including `git` commands, `uv` commands, etc. Compounding requires manual approval and defeats autonomous execution.

## Step 1: Detect context and ensure correct branch

Run `git branch --show-current` and check the working directory.

**Case A — on `main`:**
1. Derive a branch name from the task (see "Branch naming" below).
2. Run `git fetch origin main` then `git checkout -b <branch-name> origin/main`.

**Case B — on a feature branch or in a worktree:**
First, verify the branch is **related to the current task** — check the branch name against the ticket number, slug, or task description. If it matches, use the current branch. If the branch is for **unrelated work**, treat as Case A — create a new branch from `origin/main`. Never pile unrelated changes onto an existing branch.

### Branch naming (for Case A only)

- **Linear ticket** (e.g. `MER-381`): `claude/mer-381-<short-description>`
- **Other task**: `claude/<type>-<short-description>` where `<type>` is `feat`, `fix`, or `chore`

## Step 2: Get context

If the argument looks like a Linear ticket (e.g. `MER-381`), fetch the ticket details using the Linear MCP tools. Use the ticket title and description to plan the implementation.

Otherwise, use the argument as the task specification directly.

## Step 3: Do the work

Carry out the task described in the arguments. Follow all project conventions from CLAUDE.md and `.claude/rules/`.

## Step 4: Lint

Run `/lint` to check the changes before committing. Fix any issues found.

## Step 5: Commit and push

Stage changes with `git add`, then commit with a descriptive message. Push the branch with `git push -u origin <branch-name>`.

**Important**: Never chain `git add` and `git commit` in the same bash call (pre-commit hooks hold the index lock).

## Step 6: Open PR as ready for review

Run `gh pr create` **without** `--draft`. This overrides the general CLAUDE.md guidance — `/send` tasks are focused and automated, so they go straight to ready for review. Opening as ready triggers the Claude auto-review bot.

PR title must start with a bracketed prefix:
- Linear ticket: `[MER-381] Short description`
- Other: `[feat]`, `[fix]`, or `[chore]` followed by a short description

**Important**: Write the PR body to a uniquely-named temp file using the `Write` tool and use `--body-file` instead of inline `--body`. Use the ticket number or branch slug in the filename (e.g. `/tmp/pr-body-mer-381.md`) to avoid collisions with concurrent worktrees. Both inline `--body` and bash heredocs with `#` markdown headings trigger Claude Code's permission check ("quoted newline followed by #-prefixed line") and require manual approval. The `Write` tool is auto-allowed for `/tmp/pr-body-*` paths via the skill's `allowed-tools`.

```
Write tool: /tmp/pr-body-mer-381.md
gh pr create --title "[feat] ..." --body-file /tmp/pr-body-mer-381.md
```

## Step 7: Wait for the review bot

Run `gh pr checks <number> --watch` to block until all checks (including `claude-review`) complete. If checks hang for more than 10 minutes, report the status to the user and ask how to proceed. Then fetch comments with `gh api repos/{owner}/{repo}/issues/{number}/comments`.

## Step 8: Address review feedback

Run `/address-pr-feedback` to read and address the bot's review comments. No need to wait for a follow-up review — the user will do the final review themselves.

## Step 9: Report back

Summarize what was done and share the PR URL.
