---
name: copy-db
description: Copy the dev SQLite database from the main repo into the current worktree.
allowedTools:
  - Bash(cp *)
  - Bash(mkdir *)
  - Bash(git worktree list *)
---

# Copy dev database to worktree

Copy `merlin.db` from the main repo into the current worktree so the backend has a working database.

**Do NOT copy WAL/SHM sidecar files** (`merlin.db-wal`, `merlin.db-shm`) — these are SQLite runtime artifacts that get auto-generated. Copying them can cause issues if the source DB has an active connection.

## Steps

1. **Detect worktree context** — check if the current working directory contains `.claude/worktrees/`. If not, report an error: this skill only works inside a worktree.

2. **Determine the worktree root** — extract the worktree path from the working directory (the directory inside `.claude/worktrees/<name>/`).

3. **Determine the main repo root** — run `git worktree list --porcelain` and extract the first worktree path (the line starting with `worktree `). This is always the main working tree. Use this as `<main-repo>` for all source paths.

4. **Check if DB already exists** — if `<worktree>/backend/data/merlin.db` already exists, skip with a message: "Database already exists in worktree, skipping copy." This is a no-op.

5. **Ensure destination directory exists** — run `mkdir -p <worktree>/backend/data/`.

6. **Copy the database** — copy only the main DB file from the main repo:
   - Source: `<main-repo>/backend/data/merlin.db`
   - Destination: `<worktree>/backend/data/merlin.db`

7. **Confirm** — report that the database was copied successfully.
