---
name: validate-ui
description: Drive the pages a change touches in the local dev app with the Playwright MCP and record what each page rendered. Signs in as the auth provider's test user for gated routes.
---

# Skill: /validate-ui [pages | PR]

Type checks and unit tests prove the code compiles and the fixtures agree. Only the running app proves a page still shows the value. This skill runs the scripted user tour against the real dev frontend and database, then drives any page the diff touches that the tour does not, and records what rendered.

`/send` runs it as its validation step on every PR; it also runs on its own ("did you click around?").

The tour is `frontend/e2e/tour.e2e.ts` (`bun run e2e` from `frontend/`), a Playwright suite that needs no Claude in the loop: it signs in as the test user, then clicks from the home page the way a user does through each product area (search, detail page, related items, compare) and the catalog pages, asserting non-empty data cells, no `null`/`undefined`/`NaN` text, at least one figure per detail page, and no console errors from our code.

## Arguments

- A list of routes to drive, or a PR number whose diff decides the routes, or nothing (use the current branch's diff against `origin/main`).

## Step 1: Servers

Invoke `/dev-servers`; note the ports it reports.

## Step 2: Run the tour

From `frontend/`: `bun run e2e`. It reads `E2E_BASE_URL` (default `http://localhost:3000`); pass the frontend port `/dev-servers` reported if it differs. The HTML report lands in `frontend/scratch/playwright-report` (`bun run e2e:report` opens it); traces and screenshots for failures in `frontend/scratch/test-results`.

A failing step is a defect in the app until shown otherwise: read the trace, fix the app, commit as `chore: ui fixes`, push, and re-run. Fix the tour itself only when the app is right and the selector went stale, and say so in the record.

## Step 3: Drive what the tour does not cover

List the routes the diff can affect that the tour does not walk (a new page, a changed figure family, an admin or content page). Drive those with the Playwright MCP against `localhost`, signed in as the same test user through the real sign-in form. The credentials are in `frontend/e2e/test-user.ts`; the dev auth instance runs in test mode, so the test address takes a fixed verification code and no email is sent. Read the accessibility snapshot and the DOM (`browser_snapshot`, `browser_evaluate` on `getBoundingClientRect` / computed styles for layout), not a screenshot. Screenshots go under `scratch/` (gitignored), passed as `filename: scratch/<name>.png`.

For each page record the values that rendered, named (`<the actual values you saw>`), the console error count, and any redirect, blank cell, `null` / `undefined` in text, or missing figure. Third-party widgets may log console errors of their own on auth pages; everything else counts.

**On a Fable session, delegate the MCP driving to an Opus subagent** (the model-roles rule in CLAUDE.md covers implementation bodies; browser driving is the same trade). Give it the diff summary, the page list, and where the credentials live, and ask for the per-page record. The verdict, any fix, and the PR comment stay in the main loop.

## Step 4: Extend the tour when a page is new

If the diff adds a page or tool a user reaches by clicking, add the click path to `tour.e2e.ts` in the same PR, so the next run covers it without this skill.

## Step 5: Record

Post the record as one PR comment when a PR exists (fold it into an earlier `/send` comment if that one has not been posted yet):

```markdown
**UI validation** (local dev servers on this branch)

- Tour: `bun run e2e` N passed / M failed, <duration>
- `/<route>` (beyond the tour): what rendered, with values; console: N errors
- ...
- Not driven: <routes>: <reason>
```

Then stop the dev servers unless the user is about to click around.
