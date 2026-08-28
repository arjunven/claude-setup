# claude-setup
A snapshot of my Claude Code setup.

**This isn't meant to be copied wholesale. Treat it as inspiration.** It's an example of one setup, very specific to how I work. Use it as a reference for how you might build up your own rules and skills; some pieces will lift straight across, most of it you'll want to adapt, and you should add it piece by piece. None of it was made in one shot either. Whenever you catch yourself doing some manual, repetitive thing over and over, ask Claude to make a skill for it. I'd say about 50% of my PRs have a small tweak or a new skill added to the `.claude` folder. I don't make separate PRs just for Claude stuff, I roll it in as I go.

Some skills are specific to my projects; however, the following are the highlights that are worth copying. Project-specific stuff (domain vocabulary, real paths, real ticket numbers) has been scrubbed or replaced with placeholders like `PROJ-123` and `app.db`.

## A note on worktrees

Most of this setup is built around git worktrees, and a lot of people haven't used them. A branch is a pointer to a commit; you only ever have one checked out in a given directory, so switching branches means stashing or committing whatever you're in the middle of. A **worktree** is a second (third, fourth...) working directory attached to the same repo, each with its own branch checked out. So you can have four Claude Code sessions running in four terminal tabs, each in its own worktree on its own branch, none of them stepping on each other's files, all sharing one `.git`.

That's the whole trick here: `/worktree` spins one up per task, `/send` does the work inside it, and `/exit-worktree` removes it once the PR is merged. Claude Code has native support for this (the `EnterWorktree` / `ExitWorktree` tools, and a `.claude/worktrees/` convention), which the skills use when available.

Reading:
- [`git worktree` docs](https://git-scm.com/docs/git-worktree)
- [Claude Code: run parallel sessions with git worktrees](https://code.claude.com/docs/en/common-workflows#run-parallel-claude-code-sessions-with-git-worktrees)

Some notes:
- Skills are composable, so you'll notice that some of the larger skills like `/send` or `/worktree` actually call the smaller skills like `/address-pr-feedback` or `/lint` within them.
- This is nice because you can just use the smaller skills as you need, or Claude will use them as it needs them ad hoc. When it's called in a larger skill, it's just like little pieces of a recipe.
- I have Linear plumbed right into Claude via the [Linear MCP](https://linear.app/docs/mcp), and I find it really nice as a place to compose tickets with Claude. I'll talk through an issue and figure out the architecture. Claude will actually write the whole plan to a Linear ticket, and I can choose to `/send` that Linear ticket right now or save it for later. At a future date, when that Linear ticket is relevant, I can use the `/worktree` skill or the `/send` skill to just implement that ticket. The whole plan is already fleshed out in there.
- Also, another plug: I started using [Wispr Flow](https://wisprflow.ai/), and it's kind of nuts. You can just talk to Claude, and it's really good at voice dictation. (Wrote most of this with Wispr Flow.)
- I run Claude Code in a CLI in a terminal called [cmux](https://cmux.com/), one tab per worktree. Also recommend [ghostty](https://ghostty.org/).

## What's in here

```
.claude/
  CLAUDE.md          # the general-purpose half of my project CLAUDE.md
  rules/             # path-scoped rules (loaded only when touching matching files)
  skills/            # the slash commands below
```

## The big picture: `/worktree` → `/send`

`/worktree` is my most used skill. I open a new terminal tab, pass it a Linear ticket number or a description of a bug (plus a screenshot if useful), and it spins up an isolated git worktree, installs deps, and hands off to `/send`. `/send` does the work, lints, commits, opens the PR as ready-for-review, **reviews its own PR in the same session**, runs a simplify pass, validates the UI in the running dev app, waits for CI, and reports back. By the time I look at the PR it's already been through a review loop.

The review used to run as a GitHub Actions workflow on every PR (that's what earlier versions of this repo shipped). I dropped that in favour of reviewing inside `/send`: it's faster, it can apply its own fixes to the working tree before the PR is even opened, and it doesn't burn CI minutes re-reviewing every push.

## `/send`
Implement a task, ship it as a PR, and review it before I ever see it. The steps:

1. Find or create the right branch (never on `main`).
2. Fetch the Linear ticket if given one.
3. Do the work, `/lint`, commit, push, open PR (ready for review, not draft).
4. **Review tier by risk** — light diffs get the built-in `/code-review` at a level Claude picks; risky diffs (auth, billing, migrations, core domain correctness) get a heavier review. In my repo that's a private `/deep-review` skill that isn't included here; the built-in `/code-review` at `high` is the drop-in. Findings are applied as a `chore: review fixes` commit and summarised in a PR comment.
5. **`/simplify` polish pass** — the review tiers only hunt correctness bugs, so duplicated helpers, dead exports, and comments that restate the code survive a clean review. `/simplify` catches those. Also committed + summarised.
6. **`/validate-ui`** — drive the pages the diff touches in the real dev app.
7. Wait for CI, report back.

## `/worktree`
Create a worktree, `uv sync` + `bun install`, then `/send`. Pass a Linear ticket or a description.

## `/setup-worktree`
A fresh worktree only has what's committed, so gitignored files like `.env` and a local dev database aren't there. This copies them over from the main checkout (idempotent, so it's safe to call repeatedly). `/dev-servers` calls it automatically.

## `/exit-worktree`
Verifies the PR is merged or closed, then removes the worktree and its branch.

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
Path-scoped rules that only load when Claude touches matching files. `frontend-conventions.md` is included as an example of the shape: Next.js gotchas (rendering mode, `loading.tsx` vs `<head>`, noindex/canonical), component style, feature-folder layout.
