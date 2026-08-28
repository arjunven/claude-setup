---
name: setup-worktree
description: Bootstrap a worktree by copying the dev SQLite database and `.env` file from the main repo. Most backend scripts need both.
allowedTools:
  - Bash(cp *)
  - Bash(mkdir *)
  - Bash(git worktree list *)
  - Bash(ls *)
---

# Bootstrap worktree (DB + .env)

Copy `app.db` and `.env` from the main repo into the current worktree so the backend has a working database and the env vars (DB URL, object-storage creds, external API keys) needed by most scripts. Both files are gitignored, so the default worktree-creation flows (CLI `worktree` skill, GUI auto-worktree) don't include them.

**Do NOT copy WAL/SHM sidecar files** (`app.db-wal`, `app.db-shm`) — these are SQLite runtime artifacts that get auto-generated. Copying them can cause issues if the source DB has an active connection.

## Steps

1. **Detect worktree context** — check if the current working directory contains `.claude/worktrees/`. If not, report an error: this skill only works inside a worktree.

2. **Determine the worktree root** — extract the worktree path from the working directory (the directory inside `.claude/worktrees/<name>/`).

3. **Determine the main repo root** — run `git worktree list --porcelain` and extract the first worktree path (the line starting with `worktree `). This is always the main working tree. Use this as `<main-repo>` for all source paths.

4. **Copy `app.db` (idempotent)**:
   - If `<worktree>/backend/data/app.db` already exists, skip with a message: "Database already exists in worktree, skipping copy."
   - Otherwise: `mkdir -p <worktree>/backend/data/` then `cp <main-repo>/backend/data/app.db <worktree>/backend/data/app.db`. Copy ONLY the `.db` file — never the `-wal` / `-shm` sidecars.

5. **Copy `.env` (idempotent)**:
   - If `<worktree>/.env` already exists, skip with a message: "`.env` already exists in worktree, skipping copy."
   - If `<main-repo>/.env` does not exist, warn: "No `.env` found in main repo at `<main-repo>/.env` — skipping (you may need to create one from `.env.example`)." Do not error; the user may not have set one up yet.
   - Otherwise: `cp <main-repo>/.env <worktree>/.env`.

6. **Confirm** — report what was copied (DB, `.env`, both, neither). Mention if either was already present so the user knows the skill was a no-op for that file.
