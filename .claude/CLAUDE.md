(This is not my whole Claude Md but rather some general stuff that might be nice for others to copy)

## Worktree awareness

- **Detect worktree context** — if the working directory contains `.claude/worktrees/`, you are in a worktree. The worktree root is the directory inside `.claude/worktrees/<name>/`.
- **When working in a worktree, always use the worktree path for all operations** — file reads, searches (Glob, Grep), edits, and shell commands must target the worktree directory, never the main repo root. The worktree contains the WIP changes; the main repo root does not.
- **CRITICAL: Check if a worktree exists before writing any code** — at the start of every task, check the git status snapshot and working directory. If the branch name suggests a worktree (e.g. `worktree-*`), `cd` into the worktree directory (`/.claude/worktrees/<name>/`) **before** reading files, making edits, or running commands. Never write code in the main repo when a worktree is active — all your changes will end up in the wrong place.
- **After plan mode exits into a worktree branch, immediately `cd` to the worktree** — do NOT stay in the main repo root. The plan was created for work in the worktree.

## Branch discipline

- **NEVER write code on `main`** — not a single file edit, not even "I'll move it later." Before touching any code, you MUST be on the correct branch. This is a hard prerequisite for all work.
- **First action of every task: find the right branch** — run `git branch --show-current`. Then:
  1. **If on `main`**: run `git worktree list` and `git branch -a` to find an existing worktree or branch related to the task (match on ticket numbers, feature names, or keywords from the task description). If a worktree exists, `cd` to it. If a branch exists, `git checkout` it. Only create a new feature branch from `origin/main` if nothing related exists.
  2. **If on a feature branch**: verify it is **related to the current task** before proceeding. Check the branch name, ticket number, and recent commits — if the branch is for unrelated work (e.g. on `claude/mer-444-mosfet-compare` but the task is about adding a new skill), treat it the same as being on `main`: create a new branch or worktree for the new task. Never pile unrelated changes onto an existing feature branch.
- **In a worktree, use the worktree branch** — don't create a new branch; the worktree was set up with one already.
- **Prefix Claude-created branches with `claude/`** (e.g. `claude/fix-login-bug`).

## PR and commit conventions

- **PR titles must start with a bracketed prefix** — Linear ticket (`[MER-123]`) or [Conventional Commits](https://www.conventionalcommits.org/) type (`[feat]`/`[fix]`/`[chore]`/etc.). The PR title becomes the squash commit message.
- Individual commits don't need the prefix — they get squashed.
- **Use `--body-file` for PR bodies** — write the body to a uniquely-named temp file (e.g. `/tmp/pr-body-mer-398.md`) and pass `--body-file` instead of inline `--body`. Use the ticket number or branch slug in the filename to avoid collisions with concurrent worktrees. Inline body strings with `#` markdown headings trigger Claude Code's permission check and require manual approval.
