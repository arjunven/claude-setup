---
name: dev-servers
description: Spin up backend and frontend dev servers in the background. Auto-detects worktree context.
allowedTools:
  - Bash(uv run dev *)
---

# Start dev servers

Spin up the backend and frontend development servers in the background for the current context (worktree or main repo).

## Steps

1. **Detect context** — check if the current working directory contains `.claude/worktrees/`. If so, you are in a worktree; otherwise you are in the main repo.

2. **If in a worktree, run `/copy-db` first** — invoke the `copy-db` skill to ensure the worktree has a database. This no-ops if the DB already exists. Skip this step when running from the main repo.

3. **Read `tools/dev.py`** — understand the correct commands, flags, port handling, and how to connect the frontend to the backend. Do not hardcode commands or ports — derive them from the script. Key details:
   - Backend: `uv run dev backend` (auto-finds a free port starting from 8000)
   - Frontend: `uv run dev frontend --backend-port <port>` (connects frontend to the backend's port via `NEXT_PUBLIC_API_URL`)
   - If the backend can't bind to port 8000, it picks the next free port and logs the actual port.

4. **Start the backend** — run `uv run dev backend` using `run_in_background: true`. Monitor the initial output to capture the actual port it binds to.

5. **Check backend output** — read the backend's background task output to find the actual port (look for the "API docs available at" or port-in-use warning log lines).

6. **Start the frontend** — run `uv run dev frontend --backend-port <actual_port>` using `run_in_background: true` (separate Bash call). If the backend is on the default port 8000, the `--backend-port` flag can be omitted.

7. **Check frontend output** — read the frontend's background task output (use `TaskOutput` with `block: false` after a few seconds) to find the actual port. Next.js may pick a different port if 3000 is in use — look for the `Local:` line (e.g. `- Local: http://localhost:3002`) or the "Port 3000 is in use" warning.

8. **Report URLs** — tell the user the **actual** ports from the server output:
   - Backend API: `http://127.0.0.1:<backend_port>`
   - API docs: `http://127.0.0.1:<backend_port>/docs`
   - Frontend UI: `http://localhost:<frontend_port>`
   - Mention that both servers are running in the background and you can relay logs or errors on request.
