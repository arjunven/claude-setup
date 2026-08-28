---
name: ide-worktree
description: Open a git worktree in VS Code (or compatible IDE). Pass a worktree name/branch/slug to target a specific one, or omit to open the current worktree.
allowed-tools: Bash(code:*),Bash(cursor:*),Bash(git worktree list:*),Bash(command -v:*)
---

# Skill: /ide-worktree [name]

Open a worktree in the IDE. Resolves the target path from `git worktree list`, then launches the `code` CLI (or `cursor` if VS Code is not installed).

## Steps

1. **Resolve the target worktree path**:
   - If an argument is given, run `git worktree list --porcelain` and match against the argument in this order (stop at the first tier with a hit):
     1. Exact match on the branch name.
     2. Exact match on the last path segment (the `<name>` in `.claude/worktrees/<name>/`).
     3. Case-insensitive substring match on path or branch. If exactly one worktree matches, use it; if multiple match, list them and ask the user to disambiguate.
     4. No match — report the available worktrees and stop.
   - If no argument is given, use the current working directory's worktree:
     - If inside `.claude/worktrees/<name>/`, that is the target.
     - If in the main working tree, stop and tell the user to pass a worktree name (opening the main repo is not the purpose of this skill).

2. **Pick the IDE CLI**:
   - Prefer `code` (VS Code's CLI). Check with `command -v code`.
   - Fall back to `cursor` if `code` is not on PATH.
   - If neither is available, report the issue and stop — do not try to install anything.

3. **Open it**: Run `<cli> <absolute-worktree-path>`. This launches a new window rooted at the worktree.

4. **Confirm**: Report which worktree was opened and with which CLI.
