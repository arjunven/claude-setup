---
name: exit-worktree
description: Exit the current worktree and clean it up after verifying the associated PR has been merged or closed.
allowed-tools: ExitWorktree,Bash(cd *),Bash(git worktree:*),Bash(git branch:*),Bash(git checkout *),Bash(git status *),Bash(gh pr view:*),Bash(gh pr list:*)
---

# Skill: /exit-worktree

Exit the current worktree, switch back to `main`, and remove the worktree and its branch — but only after confirming the associated PR is done.

## Steps

1. **Detect worktree context**: Run `git worktree list` to identify the current worktree. If the current working directory is not inside a worktree (i.e. it's the main working tree), stop and tell the user there's nothing to exit.

2. **Identify the branch and worktree path**: From the worktree list, extract:
   - The worktree absolute path
   - The branch name (e.g. `worktree-mer-381-datasheet-extractor`)

3. **Find the associated PR**: Run `gh pr list --head <branch> --state all --json number,state,title --limit 1` to find the PR for this branch.
   - If no PR exists, warn the user and ask whether to proceed with cleanup anyway (the work may have been abandoned).

4. **Check PR status**:
   - If **merged** or **closed**: proceed with cleanup.
   - If **open**: stop and tell the user the PR is still open. Show the PR number and title. Ask if they want to close it and clean up, or abort.

5. **Check for uncommitted changes**: Run `git status` in the worktree. If there are uncommitted changes, warn the user and ask whether to discard them or abort.

6. **Remove the worktree**: Use the `ExitWorktree` tool with `action: "remove"`. This handles switching back to the main repo root automatically.
   - If `ExitWorktree` reports no active session (worktree was created in a prior conversation), fall back to running `git worktree remove <absolute-path>` from any directory — do NOT try to `cd` to the main repo first. If it fails due to unclean state, use `--force` after confirming with the user.

7. **Delete the local branch** (only needed if `ExitWorktree` was not available): Run `git branch -d <branch>`. If the branch hasn't been fully merged, use `git branch -D <branch>` after confirming with the user.

8. **Report**: Confirm the worktree and branch have been removed.
