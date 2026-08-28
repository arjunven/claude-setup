---
name: deep-review
description: The heavy review tier — autonomous multi-agent code review that applies its own findings, running the vendored /code-review engine at xhigh on Opus. Pass an optional diff target (defaults to this branch's commits plus any uncommitted work). Called by /send for risky diffs; light diffs get the built-in /code-review instead.
---

# Skill: /deep-review [target]

Reviews a diff with the same fan-out engine as the built-in `/code-review`: correctness finders across five angles, a cleanup finder, an independent verifier per finding location, a gap sweep, and a ranked synthesis.

Always runs at `xhigh` on Opus, and always applies its findings. There is no level argument and no `--fix` flag by design — for a review that only reports, or one at a different effort level, use the built-in `/code-review` (the light tier in `/send`, or manually at any level).

## Why this exists

The built-in `/code-review` dropped its multi-agent workflow engine in Claude Code 2.1.232: every local effort level now runs as a single background fork, and verification-grade multi-agent review moved to the billed cloud products (`/code-review ultra`, the Code Review GitHub app). `code-review-workflow.js` in this skill's directory preserves the final workflow-script revision of that engine — correctness finders, independent per-location verifiers, gap sweep, ranked synthesis — so risky diffs keep a verification-grade review that runs locally. This skill launches it through the generic Workflow tool; `/send` invokes it as the heavy review tier. See the script's header for provenance.

## Step 1: Build the review target

1. Run `git fetch origin main`. This is unconditional — a stale `origin/main` silently widens or empties the diff. If the fetch fails, say so and stop; do not review against a stale base.
2. If the caller supplied a target, use it (see the flag note below). Otherwise build the default:
   - Start from `origin/main...HEAD`.
   - Run `git status --porcelain`. If it reports **any** uncommitted or untracked files, append them to the target description explicitly, e.g. `origin/main...HEAD plus uncommitted changes (git diff HEAD) and untracked files`.

Step 2 of the engine's own scope prompt only offers the "also include uncommitted changes" clause when no target is given, and this skill always passes one — so uncommitted and untracked work is invisible unless the target names it. Naming it is what makes the skill's "reviews your changes" promise true when run outside `/send`.

**Strip any flag-shaped token before it reaches the target.** Fixing is unconditional, so `--fix` is not an option — but a caller may still type it out of habit. Passing it through verbatim yields `git diff origin/main...HEAD --fix` → `fatal: unrecognized argument`, and the stray token also rides into every finder prompt as scope guidance. Drop leading-dash tokens from the argument string; only the diff spec reaches the engine.

## Step 2: Run the engine

Invoke the **Workflow tool** with:

- `scriptPath`: the **absolute** path to `code-review-workflow.js` in this skill's directory. A bare filename does not resolve — the session cwd is the repo or worktree root, not the skill directory.
- `args`: `"xhigh <target>"` — a single string, e.g. `"xhigh origin/main...HEAD"`.

The `xhigh` prefix is required: the script parses its first token as the level and silently degrades to `high` (3 finders, no sweep, 10-finding cap) if it isn't one, folding the stray token into the target.

The workflow runs in the background; wait for its completion notification rather than polling. If the result comes back empty or malformed, read `journal.jsonl` in the run's transcript dir (path is in the tool result) to see what each agent actually returned before diagnosing.

## Step 3: Check the scope before trusting the result

**An empty scope is a failure, not a clean review.** The engine returns `summary: "No changes found to review."` with `findings: []` whenever the scope agent finds zero changed files — byte-identical in shape to a review that ran fully and found nothing. Publishing that as "no findings" claims a review that never happened.

Before reporting, check `stats.finders` in the result. If it is `0`, or the summary is `No changes found to review.`, treat the run as **not reviewed**: say so plainly, state the target that produced no diff, and do not report or post a clean result. Fix the target and re-run.

## Step 4: Report

Report the findings with the **ReportFindings** tool — one call, most-severe first, empty array only if the engine genuinely verified findings and none survived.

- Set `level` from the result's own `level` field, not a hardcoded `xhigh`. If the engine degraded to `high`, the report must say `high`.
- The engine emits `file`, `line`, `summary`, `failure_scenario`, `category`, `verdict` but no `short_summary`. Derive a `short_summary` (≤60 chars, the claim alone) per finding.

Don't also print the findings as prose; a short summary paragraph is enough.

## Step 5: Apply fixes

- Apply every **CONFIRMED** finding as a direct edit.
- Apply a **PLAUSIBLE** finding only when the fix is unambiguous and low-risk; otherwise leave it for the human.
- **`code-review-workflow.js` is frozen at upstream's final revision** (upstream stopped emitting workflow scripts, so no refresh is possible and the file is ours to maintain). Still don't patch engine defects as a side effect of a review run: report them, and change the engine only as a deliberate separate edit that keeps the five Opus/xhigh pins.
- Report which findings were applied and which were left, separating CONFIRMED from PLAUSIBLE so the caller's PR comment can do the same.
- Run `/lint` afterwards. Leave staging, committing, and pushing to the caller.

## Labeling

In any PR comment or summary, call this an **internal deep review (vendored /code-review engine)**. Never present it as the built-in `/code-review` — and never present a built-in run as this review.
