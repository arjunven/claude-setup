---
name: send
description: Implement a task and ship it — commit, push, open a PR as ready for review, run the risk-matched review tier and fix, run a /simplify polish pass, validate the change in the running frontend with /validate-ui, then post a summary. Pass a Linear ticket (e.g. PROJ-381) or a description of the task. Works from any context (worktree, feature branch, or main).
allowed-tools: Bash(gh pr create:*),Bash(gh pr checks:*),Bash(gh pr view:*),Bash(gh api:*)
---

# Skill: /send <task-or-ticket>

Implement a task, ship it as a PR ready for review, and review it in the same session — the built-in `/code-review` for light diffs, `/deep-review` for risky ones (Step 7 has the criteria).

## Shell discipline (CRITICAL)

**NEVER compound `cd` with other commands** — `cd /path && command` is FORBIDDEN. Always `cd` in one Bash call, then run the next command in a separate Bash call. This applies to ALL commands including `git` commands, `uv` commands, etc. Compounding requires manual approval and defeats autonomous execution.

## Step 1: Detect context and ensure correct branch

Run `git branch --show-current` and check the working directory.

**Case A — on `main`:**
1. Derive a branch name from the task (see "Branch naming" below).
2. Run `git fetch origin main` then `git checkout -b <branch-name> origin/main`.

**Case B — on a feature branch or in a worktree:**
First, verify the branch is **related to the current task** — check the branch name against the ticket number, slug, or task description. If it matches, use the current branch. If the branch is for **unrelated work**, treat as Case A — create a new branch from `origin/main`. Never pile unrelated changes onto an existing branch.

### Branch naming (for Case A only)

- **Linear ticket** (e.g. `PROJ-381`): `claude/proj-381-<short-description>`
- **Other task**: `claude/<type>-<short-description>` where `<type>` is `feat`, `fix`, or `chore`

## Step 2: Get context

If the argument looks like a Linear ticket (e.g. `PROJ-381`), fetch the ticket details using the Linear MCP tools. Use the ticket title and description to plan the implementation.

Otherwise, use the argument as the task specification directly.

## Step 3: Do the work

Carry out the task described in the arguments. Follow all project conventions from CLAUDE.md and `.claude/rules/`.

## Step 4: Lint

Run `/lint` to check the changes before committing. Fix any issues found.

## Step 5: Commit and push

Stage changes with `git add`, then commit with a descriptive message. Push the branch with `git push -u origin <branch-name>`.

**Important**: Never chain `git add` and `git commit` in the same bash call (pre-commit hooks hold the index lock).

## Step 6: Open PR as ready for review

Run `gh pr create` **without** `--draft` — `/send` tasks go straight to ready for review.

PR title must start with a bracketed prefix:
- Linear ticket: `[PROJ-381] Short description`
- Other: `[feat]`, `[fix]`, or `[chore]` followed by a short description

## Step 7: Review and fix

Every `/send` gets a review before it ships. First run `git fetch origin main` to refresh the merge target, then pick the tier by risk.

**Heavy — invoke `/deep-review origin/main...HEAD`** when the diff deserves it. There is no size cutoff; judge by what a missed bug would cost, not by line count. Always heavy when any of these is true:

- it touches auth, billing, paywall, or quota enforcement (`frontend/src/features/auth/`, `frontend/src/features/billing/`, webhook handlers)
- it adds or changes DB migrations, the DB access layer, or query shapes
- it changes the correctness of your core domain logic (for us: `backend/app/<core-domain>/`, its prompts, fixtures, or ground truth)
- you're unsure which tier applies

`/deep-review` runs the vendored `/code-review` engine at `xhigh` on Opus and applies findings as direct edits to the working tree — every CONFIRMED one, plus PLAUSIBLE ones whose fix is unambiguous and low-risk. Fixing is unconditional; there is no flag to pass.

**Light — invoke the built-in `/code-review` via the Skill tool** for everything else: docs, copy, small UI tweaks, `scripts/one_off/`, small contained fixes. Choose the effort level to match the diff and pass it with the target (args `<level> origin/main...HEAD`): `low` for trivial mechanical changes, `medium` as the default for small code fixes, `high` for substantial diffs that you judge still belong in the light tier. The heavy tier never takes a level — `/deep-review` is always `xhigh`. The built-in runs as a background fork; wait for its completion notification. It reports findings without fixing them, so apply the ones that hold up yourself, with the same CONFIRMED/PLAUSIBLE judgment. If the invocation is refused with `disable-model-invocation`, the session's skill registry predates the current permission state — fall back to `/deep-review` rather than skipping review.

**Pin the target so the review can't fall back to a stale base.** `origin/main...HEAD` scopes it to exactly what this branch adds. Omitting it is what lets the review go wide: a freshly-pushed branch makes the default base (`@{upstream}...HEAD`) empty, so it falls back to `git diff main...HEAD` — and in the worktree/send flow local `main` lags `origin/main` (we branch from `origin/main` but never fast-forward local `main`), so the review picks up everything merged in between and burns tokens on unrelated code.

If the review fails (e.g. times out on a very large PR), surface the failure to the user rather than working around it silently.

After the review returns:

1. Run `git status` to see what changed.
2. If files changed: run `/lint`, stage with `git add`, commit as `chore: review fixes`, push. Capture the SHA with `git rev-parse --short HEAD`.
3. Post one top-level PR comment summarizing the findings. Write the body to a tmpfile and pass it via `--body-file` (avoids shell-quoting issues with backticks and pipes):

```markdown
### Review summary

_Auto-generated by `/send` — <tier that ran>._

**Findings fixed** (commit `<sha>`):

- `path/to/file.py:42` — one-line description of the fix
- ...
```

Fill `<tier that ran>` with `internal deep review (vendored \`/code-review\` engine, xhigh)` or `built-in \`/code-review\` (<level>)`, naming the level that ran. Never present one tier as the other. List CONFIRMED and PLAUSIBLE findings separately, and note any that were reported but deliberately left unfixed (including defects inside the vendored engine, which are reported rather than patched in passing).

If the review ran and no findings survived, skip the commit and post instead: `**Review**: no findings (<tier>).`

**Never post that line for a review that didn't happen.** If the review reports an empty scope — no changed files found for the target — that is a failure, not a clean review. Surface it to the user with the target that produced no diff and fix it before posting anything; a green "no findings" on an unreviewed PR is worse than no comment at all.

## Step 8: Polish pass with /simplify

After the review commit is pushed, run a simplify pass on the files this branch changed. This runs on **every** `/send`, both tiers.

It exists because the review tiers deliberately don't cover this. `/deep-review` and `/code-review` hunt correctness bugs, and their verifiers refute anything that isn't one, so duplicated helpers, dead exports, aliases-for-aliases, and comments that restate the annotation all survive a clean review. `/simplify` runs four agents (reuse, simplification, efficiency, altitude) that look for exactly that.

Invoke `/simplify` via the Skill tool, passing the files changed on this branch as args (`git diff --name-only origin/main...HEAD`). Don't reach for `/deep-review` or `/code-review` here; different job.

The agents modify files in place but do not commit. After it returns:

1. Run `git status` to see what changed.
2. If files changed: run `/lint`, `git add` the modified paths, commit as `chore: simplify`, push, then capture the SHA with `git rev-parse --short HEAD`.
3. If nothing changed: skip the commit and say "no findings" in Step 11.

**Skip a finding whose fix would change rendered output or behavior.** A polish pass must not move visible copy. Report those to the user instead, with the before/after, so they can decide deliberately.

Post the findings as a second PR comment (or fold them into Step 7's comment if that one hasn't been posted yet):

```markdown
**Simplify pass** (commit `<sha>`):

- `path/to/file.ts:42` — one-line description of the change
- ...
```

If findings were skipped, list them under a **Reported but deliberately skipped** heading with the reason.

## Step 9: Validate in the running frontend

Invoke `/validate-ui`; skip it only for changes with no rendered surface (CI config, `scripts/one_off/`, backend-only tooling) and say so in Step 11.

## Step 10: Wait for CI checks

Run `gh pr checks <number> --watch` to block until all CI checks complete. If checks hang for more than 10 minutes, report the status to the user and ask how to proceed.

## Step 11: Report back

Summarize what was done and share the PR URL.
