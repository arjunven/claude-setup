# claude-setup
A snapshot of my Claude Code setup.

Some skills are specific to my projects; however, the following are the highlights that are worth copying. Project-specific stuff (domain vocabulary, real paths, real ticket numbers) has been scrubbed or replaced with placeholders like `PROJ-123` and `app.db`.

Some notes:
- Skills are composable, so you'll notice that some of the larger skills like `/send` or `/worktree` actually call the smaller skills like `/address-pr-feedback` or `/lint` within them.
- This is nice because you can just use the smaller skills as you need, or Claude will use them as it needs them ad hoc. When it's called in a larger skill, it's just like little pieces of a recipe.
- In general, these were not made all in one shot, and I don't actually recommend you copying all of these in one shot. I think it's best to just add little bits and pieces as you go, where you're like, "Oh, I do this manual, repetitive thing over and over" just ask Claude to make a skill. I would say like 50% of my PRs have a small tweak or a new skill added to the .claude folder. I don't try to make separate PRs just for claude stuff. Just roll it in as you go.
- I have Linear plumbed right into Claude via the [Linear MCP](https://linear.app/docs/mcp), and I find it really nice as a place to compose tickets with Claude. I'll talk through an issue and figure out the architecture. Claude will actually write the whole plan to a Linear ticket, and I can choose to `/send` that Linear ticket right now or save it for later. At a future date, when that Linear ticket is relevant, I can use the `/worktree` skill or the `/send` skill to just implement that ticket. The whole plan is already fleshed out in there.
- Also, another plug: I started using [Wispr Flow](https://wisprflow.ai/), and it's kind of nuts. You can just talk to Claude, and it's really good at voice dictation. (Wrote most of this with Wispr Flow.)
- I run Claude Code in a CLI in a terminal called [cmux](https://cmux.com/), one tab per worktree. Also recommend [ghostty](https://ghostty.org/).

## What's in here

```
.claude/
  CLAUDE.md          # the general-purpose half of my project CLAUDE.md
  settings.json      # SessionStart hooks + enabled plugins
  rules/             # path-scoped rules (loaded only when touching matching files)
  skills/            # the slash commands below
.github/workflows/
  claude.yml         # @claude mention handler for issues / PR comments
```

## The big picture: `/worktree` → `/send`

`/worktree` is my most used skill. I open a new terminal tab, pass it a Linear ticket number or a description of a bug (plus a screenshot if useful), and it spins up an isolated git worktree, installs deps, and hands off to `/send`. `/send` does the work, lints, commits, opens the PR as ready-for-review, **reviews its own PR in the same session**, runs a simplify pass, validates the UI in the running dev app, waits for CI, and reports back. By the time I look at the PR it's already been through a review loop.

The review used to run as a GitHub Actions workflow on every PR (that's what earlier versions of this repo shipped). I dropped that in favour of reviewing inside `/send`: it's faster, it can apply its own fixes to the working tree before the PR is even opened, and it doesn't burn CI minutes re-reviewing every push.

## `/send`
Implement a task, ship it as a PR, and review it before I ever see it. The steps:

1. Find or create the right branch (never on `main`).
2. Fetch the Linear ticket if given one.
3. Do the work, `/lint`, commit, push, open PR (ready for review, not draft).
4. **Review tier by risk** — light diffs get the built-in `/code-review` at a level Claude picks; risky diffs (auth, billing, migrations, core domain correctness) get `/deep-review`. The criteria live in the skill file. Findings are applied as a `chore: review fixes` commit and summarised in a PR comment.
5. **`/simplify` polish pass** — the review tiers only hunt correctness bugs, so duplicated helpers, dead exports, and comments that restate the code survive a clean review. `/simplify` catches those. Also committed + summarised.
6. **`/validate-ui`** — drive the pages the diff touches in the real dev app.
7. Wait for CI, report back.

## `/deep-review`
The heavy review tier. Claude Code's built-in `/code-review` used to run a multi-agent workflow (scope → parallel finders → an independent verifier per finding → gap sweep → ranked synthesis) at high effort levels; upstream moved that to the billed cloud review and the local levels now run as a single fork. I vendored the last workflow-script revision of that engine (`code-review-workflow.js`) and pinned every agent to Opus at xhigh, so risky diffs still get a verification-grade review locally. The skill launches it through the `Workflow` tool and applies CONFIRMED findings (plus low-risk PLAUSIBLE ones) directly.

## `/worktree`
Create a worktree, `uv sync` + `bun install`, then `/send`. Pass a Linear ticket or a description.

## `/setup-worktree`
Worktrees don't get gitignored files. This copies the dev SQLite DB and `.env` from the main repo into the worktree (idempotent, never copies the `-wal`/`-shm` sidecars). `/dev-servers` calls it automatically.

## `/exit-worktree`
Verifies the PR is merged or closed, then removes the worktree and its branch.

## `/ide-worktree`
Open a worktree in VS Code / Cursor by name, branch, or substring.

## `/dev-servers`
Spin up backend + frontend in the background, check the DB is on the latest migration, and report the actual ports they bound to (not the ones you hoped for).

## `/validate-ui`
Type checks prove the code compiles; only the running app proves the page still renders the value. Runs a scripted Playwright "tour" (`bun run e2e`) that clicks through the app the way a user does, then drives any page the diff touches that the tour doesn't cover with the Playwright MCP, and posts a record of what rendered on the PR.

## `/address-pr-feedback`
Pulls in all feedback from a PR (conversation, inline threads, and the `/send` review summary), classifies each item as Actionable / Question / Affirmation, **discusses the Questions before editing anything** (the most common failure mode was jumping straight to implementation when I wanted to talk it through), fixes the actionable ones, and posts a status table on the PR. Has GraphQL pagination notes for big PRs where the REST API hides thread resolution state.

## `/lint`
Runs the pre-commit hooks on changed files so Claude finds linting issues before actually committing. For me that's Ruff and ty on the Python backend and biome + tsc on the TypeScript frontend.

## `/rebase`
With many parallel worktrees, it's nice to be able to rebase the current one onto the latest `main`. Automatically fetches, rebases, resolves conflicts, and force-pushes with lease.

## `.claude/rules/`
Path-scoped rules that only load when Claude touches matching files. Included as examples of the shape:
- `testing-guidelines.md` — where tests live (mirror the package tree), what not to test, how to assert.
- `frontend-conventions.md` — Next.js gotchas (rendering mode, `loading.tsx` vs `<head>`, noindex/canonical), component style, feature-folder layout.
- `orm-serialization.md` — the SQLAlchemy JSON `"null"`-string gotcha.
- `pydantic-model-updates.md` — deep-copy-then-construct-the-leaf instead of mutating nested models.

## `settings.json`
Two `SessionStart` hooks (install git hooks, activate the Python venv) and the official plugins I keep on: `commit-commands`, `frontend-design`, `code-simplifier`, `claude-md-management`, `typescript-lsp`, `security-guidance`, `playwright`.

## Workflows Setup

The only workflow left is `claude.yml`, the `@claude` mention handler for issues and PR comments (auto-review moved into `/send`, see above).

I set it up using the OAuth token method described here: https://github.com/anthropics/claude-code-action/blob/main/docs/setup.md

> Or CLAUDE_CODE_OAUTH_TOKEN for OAuth token authentication (Pro and Max users can generate this by running claude setup-token locally)

- The `/install-github-app` described in the [Claude Code docs](https://code.claude.com/docs/en/github-actions#claude-code-github-actions) appeared to be super broken when I set this up.
  - I filed a GitHub issue about it here: https://github.com/anthropics/claude-code/issues/26227
  - The only thing it was good for is that it did walk me through the OAuth setup... but then I had to go modify the workflow files manually to follow the [examples on the Claude Code GitHub Actions repo](https://github.com/anthropics/claude-code-action/blob/main/docs/solutions.md#automatic-pr-code-review).
