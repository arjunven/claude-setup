---
name: rebase
description: Rebase the current branch onto origin/main, handling fetch and conflict resolution.
allowed-tools: Bash(git fetch:*),Bash(git rebase:*),Bash(git add:*),Bash(git push:*)
---

# Rebase onto origin/main

Rebase the current branch onto the latest `origin/main`.

## Steps

1. **Preflight check**: Run `git status` to confirm the working tree is clean (no staged, unstaged, or untracked changes). If the tree is dirty, stop and ask the user to commit or stash first.

2. **Fetch**: Run `git fetch origin main` to get the latest remote main.

3. **Rebase**: Run `git rebase origin/main`.

4. **Handle conflicts**: If the rebase stops due to conflicts:
   - List the conflicting files.
   - For each file, read the conflict markers and resolve them sensibly — prefer the current branch's intent while incorporating upstream changes.
   - Stage the resolved files with `git add`.
   - Continue with `git rebase --continue`.
   - Repeat until the rebase completes or ask the user if a conflict is ambiguous.
   - If the user wants to cancel the rebase entirely, run `git rebase --abort` to restore the branch to its pre-rebase state, then stop.

5. **Push**: If the branch has a remote tracking branch, automatically run `git push --force-with-lease` to update the remote.

6. **Report result**:
   - How many commits were replayed.
   - Whether any conflicts were resolved (and how).
   - Whether the force-push succeeded.
