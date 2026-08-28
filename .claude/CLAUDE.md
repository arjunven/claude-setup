(This is not my whole CLAUDE.md but rather the general stuff that might be nice for others to copy. Project-specific rules, domain vocabulary, and anything tied to my codebase are left out; paths like `backend/app/` and `frontend/src/` are placeholders for a Python/FastAPI + Next.js monorepo.)

# Code Standards (Backend Python)

- **Use one timing helper** for measuring durations, not manual `time.time()` scattered across modules.
- **Use `typer`** for new CLI scripts, not `argparse`.
- **Use the project logger** (loguru-based, `{}` format not `%s`). Never use `print()` or stdlib `logging`. `typer.echo()` is allowed **only** in `typer` CLI tools, and only for messages the CLI itself speaks to the operator — interactive menus, selection prompts, confirmation panels, "next step" hints. Everything else still uses `logger`.
- **`logger.opt(lazy=True)` when building the log argument is expensive.** `{}`-style defers `str.format()`, not the argument — Python evaluates it before `logger.debug` is entered, so `logger.debug("cfg: {}", cfg.model_dump_json(indent=2))` still serializes the whole payload on every call and throws it away at INFO. Keep plain `{}` for values that are already cheap (an id, a count) — that is nearly everywhere — and reach for `opt(lazy=True)` with a callable only when producing the value costs something: `model_dump_json`, `json.dumps`, a join over a large list.
- **No magic strings** — use module-level constants (e.g. `_SOURCE_VENDOR = "vendor"`).
- **Always use `datetime.now(UTC)`** (import `UTC` from `datetime`) for timezone-aware timestamps — never `datetime.utcnow()` (deprecated, naive).
- **Never use `assert` in production code** — `assert` is stripped by `python -O`, so the check silently disappears. Use `raise ValueError`/`TypeError` for internal preconditions, or validate at the API boundary. (`assert` is fine in tests.)
- **`experiments/` is POC code** — don't enforce production patterns there.
- **One-off data-mend / migration scripts go in `scripts/one_off/`** — throwaway or rarely-rerun scripts that fix or migrate committed data live there, not beside the durable CLI tools in `scripts/`. They're held to lighter standards: production patterns and test coverage aren't required, and they can be pruned once the migration has run. Still keep basic hygiene — use the project logger and `typer`, default any destructive operation to a dry-run (`--apply` to write), and commit the script so the logic is reproducible rather than stranded in a chat session. **Make it end-to-end (whole fix, one script).** If the data lives in two places (raw source-of-truth rows *and* a derived row), mend both in the one script. A partial mend that needs a second command to land is a footgun.
- **Use `PROJECT_ROOT` for file paths** — never bare relative paths like `Path("backend/...")`; they break when the tool is invoked from a different working directory.
- **No sync DB calls inside `async def` on the production hot path** — a sync DB manager called directly inside an `async` handler blocks the event loop. Wrap every call: `await asyncio.to_thread(db_manager.<method>, ...)`. Enforce it with a test that scans the hot-path directories for the pattern, so the rule can't silently regress.

- **No metaphor jargon in code, comments, docstrings, PR bodies, or chat.** Banned: "load-bearing", "actually", "the sharp edge", "here's the shape of it", "hatch", "escape hatch", em dashes as a rhetorical device. These read as filler and hide the concrete claim. Say what the thing does in plain subject-verb-object: not "this shape is load-bearing for the on-disk JSON" but "this shape defines the on-disk JSON". If a sentence loses nothing when the metaphor is removed, remove it.

- **No diff-referencing comments / docstrings** — comments must make sense to someone reading the file in isolation, not someone reading the patch that introduced them. Banned phrasings: "X was dropped", "no longer Y", "previously Z", "moved here from W", "renamed from R". If the only way to understand a comment is to know what the code looked like before, the comment is for the diff, not the file — delete it. State the current behavior and why it is the way it is.

- **Match fix scope to feedback scope.** When the user points out a small concrete issue (a redundant call, a magic number, a private-but-mutated attribute), the implicit ask is "remove that thing", not "redesign the surrounding code so the thing isn't needed". Apply the smallest local fix. Don't introduce new fields, new helpers, new control-flow paths, or new abstractions to address a problem with a 5-line fix. If the smallest fix feels ugly, surface that and ask before going bigger.

- **Match docstring style to the codebase, not to Sphinx defaults.** This project uses Google-style docstrings (`Args:` / `Returns:` headers with indented descriptions) and plain backticks for code references. Don't use reST/Sphinx roles like `:func:` / `:mod:` / `:class:` in docstrings or comments — they don't render as anything and they aren't standard Python. When unsure, grep the existing utility modules to see what real project code does.

- **Don't restate what well-named code already says.** Variable names, type annotations, and `if TYPE_CHECKING:` blocks are self-evident — a comment that paraphrases them is noise. Only write a comment when the *why* (a constraint, an invariant, a subtle interaction, a workaround for a specific bug) won't be obvious from reading the file. If you find yourself writing more than two sentences of comment for a 5-line function or a 1-field dataclass, the comment is too long.

- **Before writing a helper, inventory what the module already has.** Reasoning from first principles about what a fix "should" look like produces a plausible helper that duplicates — usually worse — one already sitting a module over, calibrated on real data. Grep the sibling modules for the primitive before writing it; a constant that already exists encodes calibration your fresh version does not have.

- **Porting between `experiments/` and production: map helpers to what already exists — don't carry POC primitives over blind.** `experiments/` is exempt from reuse standards, so it accrues private helpers that production may already provide. Before porting an experiment module into `backend/app/`, run an exploration pass (an `Explore` / `general-purpose` agent) to inventory the relevant production utilities and map each private helper onto its production equivalent — re-express against the shared util instead of copying the POC version. The discipline runs both ways: experiment code you expect to productionize should pull from production utilities from the start, so the port is a move, not a rewrite.

---

# Model roles for subagents

- **Review agents always run on Opus — never Fable.** The heavy review tier in `/send` pins `model: "opus", effort: "xhigh"` on every worker agent. The built-in `/code-review` runs as a single background fork that inherits the session model at every effort level, which is acceptable for the light tier it serves in `/send`.
- **Don't downgrade review agents to Sonnet/Haiku** without being asked.
- **When the session model is Fable, delegate implementation bodies** (roughly >30-line diffs from an approved spec) to Opus subagents; keep judgment work (architecture, review verdicts, PR descriptions) and small direct edits in the main loop.

## Pre-authorized subagent calls

Some harnesses inject a standing directive along the lines of *"do not call the Agent tool unless the user requested it"* — the desktop app does. It does **not** override the subagent work this file prescribes. Invoking a skill, or following a rule here that calls for subagents, **is** the request. Treat these as pre-authorized, no confirmation needed:

- **The adversarial plan review** (see "Plans as HTML") — automatic, as the last step of writing any non-trivial plan.
- **The exploration pass** before porting `experiments/` code into `backend/app/`.
- **The review tier inside `/send`** (`/deep-review` or the built-in `/code-review`), and `/code-review` invocations generally.

That directive is aimed at spontaneous delegation — spawning agents for work nobody asked for. Reading it as "never spawn anything" silently drops steps this file requires. When it's genuinely ambiguous, narrate and proceed ("spawning the plan review now") rather than stopping to ask; if you do substitute an inline pass for a prescribed subagent one, say so plainly instead of letting the plan read as if the review happened.

---

# Backend owns identity, frontend owns display

The API contract is **stable machine identities plus data**, not presentation. The backend ships enum codes and field names plus the data (value, `unit`, `conditions`). Human-readable labels, symbols, titles, grouping, and ordering are the **frontend's** job, owned in a typed display registry. Never bake UI English into a Pydantic model or reuse an extraction-prompt `description` as a display label.

- **Map identity to copy in a full `Record<Identity, ...>`**, keyed by the generated union type from `@/client`, so a newly-added backend identity becomes a TypeScript error at registry-edit time instead of a silent blank at render time. This is how the set of valid identities has one source (the backend enum, via `bun run type-gen`) while the copy has one source (the registry), with the type system forcing them to stay in sync.
- **Exceptions where copy may live server-side:** copy that is itself data (CMS-authored, admin- or user-editable without a deploy), or multiple human-facing consumers needing byte-identical copy (then expose a dedicated metadata endpoint, still not labels in the data model). i18n is the forcing function: the moment you would localize, identities and keys stay on the backend and copy moves to the frontend or a translation layer.

---

# Environment Setup

- **Always use `bun`** (never `npm` or `yarn`) — `bun add`, `bun remove`, `bun run`, `bun install`, `bunx`. See `frontend/package.json` for available scripts.
- **Run frontend CLI tools from `frontend/`** — `shadcn`, `bun`, `bunx`, and similar tools must be run from the `frontend/` directory where `components.json` and `package.json` live, not from the repo root.
- **Backend uses `uv`** (not pip) — `uv lock`, `uv sync`, `uv tree`; never `pip`
- **Use `uv run`** for all Python commands — `uv run pytest`, `uv run script.py`, etc.
- **Check linting early with `/lint`** — don't wait for the commit hook to find problems.
- **Dev servers** — read `tools/dev.py` for usage. Run `uv run dev backend` and `uv run dev frontend` in separate terminals.
- **Reach for the Playwright MCP on frontend UI work** — verify layout and visual changes by driving the running app and reading the real DOM, not by guessing from screenshots. Point it at the local dev server (`localhost`), not Vercel preview URLs (those require auth). **Save any screenshots into `scratch/`, not the repo root** — pass a `filename` like `scratch/foo.png` to `browser_take_screenshot`. The scratch dirs are gitignored, so the captures never show up in `git status`.
- **Never measure production response headers with `curl` — it reports the firewall, not the app.** Vercel's bot protection answers `curl` with an **HTTP 429** whose headers read exactly like a legitimately uncached page (`cache-control: private, no-store, max-age=0`, no `x-vercel-cache`). Always check the status line before trusting any header. To measure for real, drive a browser (Playwright) and read `res.headers` from a page-context `fetch`, or script a client that carries the bypass cookies.
- **Driving a *protected* Vercel preview (PR previews behind Deployment Protection):** get the preview URL from the Vercel MCP (`get_deployment` / `list_deployments`) rather than asking. The automation bypass token is `VERCEL_AUTOMATION_BYPASS_SECRET` in the gitignored `.env` — share it that way, never by pasting it into chat. Read it at runtime and append it to the *first* navigation: `?x-vercel-protection-bypass=<token>&x-vercel-set-bypass-cookie=true`, which sets a cookie covering the rest of the session. Prefer a self-contained script that reads the token from `.env` over the Playwright MCP (the MCP would surface the token in a navigation URL).
- **Use `scratch/` for one-off helper scripts** — both repo-root `scratch/` and `frontend/scratch/` are gitignored (contents ignored; an empty `.gitkeep` keeps each dir present). Use **repo-root `scratch/`** for Python/backend one-offs so they import project modules and pick up the project's `uv` / venv automatically. Use **`frontend/scratch/`** for frontend one-offs (`bun scratch/foo.ts`) — bun resolves `node_modules`, the TypeScript config, and the local `.env` from `frontend/`. Prefer scratch over `/tmp`. Clean up scripts that were truly one-off; keep ones that are likely to be reused.

---

# Working from tickets and plans

- **Never create a Linear ticket without explicit approval.** Filing is the user's call, not a default. Work routinely surfaces things worth tracking — a deferred fix, a follow-up cleanup, a bug found mid-review — and the reflex to file each one is how a session aimed at closing one ticket ends up opening two more. Say what you found and ask; don't create and then report it. In order of preference: fold it into the current PR if it belongs there, write it into the PR description or a code comment where the next reader will actually hit it, and only then propose a ticket. **Updating** an existing ticket (correcting a stale description, adding findings) needs no approval — the rule is about creating new ones.
- **Read the ticket's comments, not just its description.** When you fetch a Linear ticket, always pull its comments too (`list_comments`) before planning or writing code. Other sessions post findings, corrections, and pre-flight verification there — often newer than the description and superseding it. Treat a later comment from the ticket author as authoritative over the original description.
- **A ticket or approved plan is a starting point, not gospel.** Tickets and plans capture intent at one moment. When discussion in the session refines, reshapes, or contradicts them, the session decision supersedes the ticket. Flag that the ticket text is now stale (and offer to update it), but never treat the staleness as a blocker.
- **Conventions go to CLAUDE.md, feature decisions go to the ticket.** A durable working rule or convention that surfaced in conversation belongs here in CLAUDE.md. A decision specific to one feature (an event schema, an API shape) belongs in that feature's ticket or in code comments — not here.
- **Never name a Linear ticket by bare ID — always add a super-short description.** In plans and in chat, write it as `PROJ-123 (crawlable /pricing page)`, never `PROJ-123` on its own, so the reader knows what the ticket is without opening it. Exempt: structured identifiers that already carry the context elsewhere — PR-title prefixes like `[PROJ-123]` and branch slugs like `worktree-proj-123-feature-slug`.

---

# Plans as HTML

When producing a plan for a non-trivial task (a feature, a refactor, a salvage/review decision), write it as a **self-contained HTML file** under `scratch/plans/<ticket-or-slug>-plan.html` (gitignored) and give the bare absolute path so it opens in a browser. Quick or throwaway plans can stay as chat markdown; reach for HTML when the plan is substantial enough to review and revisit.

- **Self-contained:** inline CSS, no external assets, opens offline with no build step.
- **Always include a sticky sidebar table of contents**, like Google Docs' outline pane — plans get long and scrolling to find a section is friction. Put a `<nav id="toc">` beside `<main>` in a two-column grid, sticky-positioned, with the current section highlighted as you scroll. **Generate it from the document's own `h2`/`h3` elements with a small inline script** rather than hand-writing the list, so it can't drift out of sync when the plan is revised. Collapse it below ~64rem viewport width and in print.
- **Metadata header up top:** ticket (linked), date, session id/link, model, repo, current branch, any worktree or branch the plan targets, and the proposed branch for the work.
- **Substance over decoration:** lead with the verdict/TL;DR, then evidence with `file:line` references, a phased step-by-step plan, scope boundaries, and open decisions/risks. Same rigor as a markdown plan; the HTML is for readability, not polish for its own sake.
- **Don't commit it:** `scratch/` is gitignored, so the plan stays a review artifact, not a tracked deliverable.
- **Adversarially review the plan before executing it — automatically, not on request.** As soon as a non-trivial plan is written, spawn a read-only review subagent (`Explore` or `general-purpose`, given an adversarial brief) tasked to *attack* it: wrong assumptions, missing or mis-ordered steps, stale `file:line` references, unstated scope creep, and risks the plan glosses over. Fold the valid findings back into the plan, then kick off the work. Treat this as the last step of producing a plan — don't jump from draft straight to implementation.
- **Show what changed between plan revisions.** When you revise a plan in place, keep a short **Revision log** at the top, just under the metadata header: one entry per revision with a label or timestamp and 1–3 bullets naming what changed and why (including what the adversarial review caught). Prefer this running summary over inline diff markup in the body, so the plan always reads as the current plan.

---

# Git Workflow

- **Never chain `git add` and `git commit` in the same bash call** — pre-commit hooks hold the index lock, and chaining causes `index.lock` race conditions. Run `git add` first, wait for it to finish, then run `git commit` separately.

## Worktree awareness

- **Detect worktree context** — if the working directory contains `.claude/worktrees/`, you are in a worktree. The worktree root is the directory inside `.claude/worktrees/<name>/`.
- **When working in a worktree, always use the worktree path for all operations** — file reads, searches (Glob, Grep), edits, and shell commands must target the worktree directory, never the main repo root. The worktree contains the WIP changes; the main repo root does not.
- **CRITICAL: Check if a worktree exists before writing any code** — at the start of every task, check the git status snapshot and working directory. If the branch name suggests a worktree (e.g. `worktree-*`), `cd` into the worktree directory (`/.claude/worktrees/<name>/`) **before** reading files, making edits, or running commands. Never write code in the main repo when a worktree is active — all your changes will end up in the wrong place.
- **After plan mode exits into a worktree branch, immediately `cd` to the worktree** — do NOT stay in the main repo root. The plan was created for work in the worktree.
- **Worktrees need `.env` and the dev DB** — most scripts require env vars and the SQLite DB. The default worktree-creation flows (CLI `worktree` skill, GUI auto-worktree) don't copy these gitignored files. Before running any backend script in a fresh worktree, run the `setup-worktree` skill — it copies both from the main repo, with idempotent skips and proper WAL/SHM handling. Don't `cp` either file manually.

## Branch discipline

- **NEVER write code on `main`** — not a single file edit, not even "I'll move it later." Before touching any code, you MUST be on the correct branch. This is a hard prerequisite for all work.
- **First action of every task: find the right branch** — run `git branch --show-current`. Then:
  1. **If on `main`**: run `git worktree list` and `git branch -a` to find an existing worktree or branch related to the task (match on ticket numbers, feature names, or keywords from the task description). If a worktree exists, `cd` to it. If a branch exists, `git checkout` it. Only create a new feature branch from `origin/main` if nothing related exists.
  2. **If on a feature branch**: verify it is **related to the current task** before proceeding. Check the branch name, ticket number, and recent commits — if the branch is for unrelated work, treat it the same as being on `main`: create a new branch or worktree for the new task. Never pile unrelated changes onto an existing feature branch.
- **In a worktree, use the worktree branch** — don't create a new branch; the worktree was set up with one already.
- **Prefix Claude-created branches with `claude/`** (e.g. `claude/fix-login-bug`).

## PR and commit conventions

- **PR titles must start with a bracketed prefix** — Linear ticket (`[PROJ-123]`) or [Conventional Commits](https://www.conventionalcommits.org/) type (`[feat]`/`[fix]`/`[chore]`/etc.). The PR title becomes the squash commit message.
- Individual commits don't need the prefix — they get squashed.
- **Two review tiers in `/send`.** Light diffs get the built-in `/code-review` at a level Claude picks, `low` through `high` — a single background fork whose findings the session applies. Risky diffs get `/deep-review` — the vendored fan-out engine, Opus-pinned at xhigh. The tier criteria live in `.claude/skills/send/SKILL.md` (Step 7) and only there; when unsure, go heavy. `/code-review ultra` stays user-triggered (billed cloud review).
- **`/send` then runs a `/simplify` polish pass** (Step 8, both tiers). The review tiers hunt correctness bugs and their verifiers refute anything that isn't one, so duplication, dead exports, and comments that restate the annotation survive a clean review; `/simplify` is what catches those. It runs once per PR, before the human review, so what gets reviewed is already polished. **`/send` then validates the PR in the running dev frontend via `/validate-ui`.**

---

# Third-party config is read-only from the agent

- **Inspect third-party backend data via the vendor CLI — read-only.** Use the vendor's CLI (`GET`-shaped calls only) to list plans, users, or config. **Never make configuration changes through the CLI** — no enable/patch/mutating calls, no plan or feature edits. Vendor config (auth provider, billing, analytics) is owned and reviewed in the dashboard by humans; an agent-driven config change can silently affect billing, gating, or production users. Don't query the vendor REST API ad-hoc with `curl` and a secret from `.env` either.

---

# Adding to CLAUDE.md vs .claude/rules/

- **CLAUDE.md** — Always loaded. Use for project-wide conventions.
- **`.claude/rules/`** — Scoped by `paths:` frontmatter. Use for domain-specific guidelines.

Prefer `.claude/rules/` with a `paths:` scope when guidance only applies to a specific area.
