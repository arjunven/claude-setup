---
name: worktree
description: Create and initialize a git worktree for isolated parallel work, then invoke /send to perform the requested task inside it. Pass a Linear ticket (e.g. MER-381) or a description of the task to do.
allowed-tools: Bash(bun install:*),Bash(uv sync:*),Bash(gh pr create:*),Bash(gh pr view:*),Bash(mkdir:*),Bash(git worktree:*)
---

# Skill: /worktree <task-or-ticket>

Create an isolated git worktree, initialize its environment, then delegate to `/send` to implement and ship the task.

## Shell discipline (CRITICAL)

**NEVER compound `cd` with other commands** — `cd /path && command` is FORBIDDEN. Always `cd` in one Bash call, then run the next command in a separate Bash call. This applies to ALL commands including `git status`, `uv sync`, `bun install`, etc. Compounding requires manual approval and defeats the purpose of autonomous worktree setup.

## Steps

1. **Set title**: Invoke `/title` immediately with a short label derived from the task argument (e.g. `MER-381 Datasheet Extractor` or `Fix Login Redirect`). Do this before any other work so the user can identify this session.

2. **Get context for naming**: If the argument looks like a Linear ticket (e.g. `MER-381`), fetch the ticket details to derive a short, descriptive slug (e.g. `mer-381-datasheet-extractor`). Otherwise, derive a slug from the task description.

3. **Name the worktree**: The worktree directory and branch should reflect the work:
   - Linear ticket: `mer-381-<short-description>` → branch `worktree-mer-381-<short-description>`
   - Conventional prefix otherwise: `feat-`, `fix-`, `chore-` followed by a short slug

4. **Create the worktree**: Use the `EnterWorktree` tool with the name determined in step 3. If `EnterWorktree` is unavailable, fall back to:
   1. `git worktree add .claude/worktrees/<name> -b worktree-<name> origin/main`
   2. `cd` into `.claude/worktrees/<name>/`

5. **Initialize the environment**:
   - `uv sync` from the worktree root — creates `.venv` and installs Python dependencies
   - `bun install` in the worktree's `frontend/` directory — installs JS dependencies

6. **Send it**: Invoke `/send <original-arguments>`. PRs must be opened as **ready for review** (not draft) — this is critical so the auto-review bot triggers immediately.

## Cleanup

The user reviews and merges PRs themselves. Leave the worktree in place — use `/exit-worktree` after the PR is merged to remove it.
