---
name: address-pr-feedback
description: Fetch PR review feedback (conversation + inline comments) and address it. Pass a PR number or URL, or omit to use the current branch's PR.
allowed-tools: Bash(gh repo view:*),Bash(gh pr view:*),Bash(gh pr comment:*),Bash(gh api:*),Bash(gh pr diff:*),Bash(git add:*),Bash(git commit:*),Bash(git push:*)
---

# Skill: /address-pr-feedback [PR]

Fetch all review feedback on a PR and address it.

## Arguments

- `[PR]` — a PR number, URL, or omit to use the current branch's PR.

## Resolve the PR number

If no PR was given, run `gh pr view --json number --jq .number` to get the current branch's PR number.

## Resolve the repo

Run `gh repo view --json nameWithOwner --jq .nameWithOwner` to get the `{owner}/{repo}` for API calls.

## Fetching feedback

Run these in parallel:

- `gh pr view <number> --comments` — conversation-level comments
- `gh api repos/{owner}/{repo}/pulls/<number>/comments --paginate` — inline review comments (includes file path, line, and diff context)
- `gh pr diff <number>` — current diff for context when addressing inline comments

Comments come from several sources, all surfaced via the same API:
- **Local `/send` review** (`/deep-review` or the built-in `/code-review`, Step 7): one top-level comment titled "Review summary", authored under the user's `gh` CLI auth, listing findings as `file:line` bullets and naming the tier that ran. The API reports the same author as manual user comments, so recognize it by that structure; manual comments are conversational.
- **Manual user comments**: the human reviewer's thoughts.
- **Other workflow bots**: Vercel preview, `@claude` mention handler, etc.

These are often the first round of feedback to address. Label the Source column in Step 1's table accordingly (`deep-review`, `@username`, `Vercel`, etc.).

### Filtering by resolution status (large PRs)

The REST `/comments` endpoint above returns **every** inline comment regardless of whether the thread is resolved or outdated. On a small PR that's fine — read the diff context, ignore comments on lines that have changed. On a large PR (50+ inline threads) you need to filter to **active** threads (not resolved, not outdated). The REST API doesn't expose that state; only GraphQL `reviewThreads` does.

When the PR has 50+ inline threads, **switch to GraphQL** for thread resolution status:

```graphql
{
  repository(owner: "<owner>", name: "<repo>") {
    pullRequest(number: <number>) {
      reviewThreads(first: 100) {
        pageInfo { hasNextPage endCursor }
        totalCount
        nodes {
          isResolved
          isOutdated
          path
          line
          comments(first: 5) { nodes { author { login } body } }
        }
      }
    }
  }
}
```

**GraphQL does NOT auto-paginate.** `first: 100` is a hard cap and the query silently drops the rest. Two things you must do:

1. **Read `totalCount` and check it against your fetched node count.** If `totalCount > 100`, you missed threads on a single-page query.
2. **Follow `pageInfo.hasNextPage` / `endCursor` and re-query with `after: <endCursor>`** until you've fetched all pages.

Worked example for a 107-thread PR:
```bash
# Page 1 — also returns hasNextPage and endCursor
gh api graphql -f query='{ ... reviewThreads(first: 100) { ... } }' > page1.json

# If hasNextPage: true, fetch page 2 with the cursor
gh api graphql -f query='{ ... reviewThreads(first: 100, after: "<endCursor>") { ... } }' > page2.json

# Verify: sum of all fetched nodes must equal totalCount
```

**Cross-check before declaring "all addressed".** Before saying you've walked every active thread, verify two things:
- The number of active threads you fetched matches what the PR's UI shows under "Unresolved comments".
- The set of `(path, line)` pairs in your fetched threads covers every comment the user has seen.

If either is off, you've missed comments. The most common failure: a 107-thread PR returned 100 nodes on a single GraphQL page and the workflow declared completion having walked 7 active threads when the actual count was 10.

## Addressing feedback

Collect **all** feedback — owner inline comments, conversation comments, and `/send` review-summary items (including non-blocking "nits" and "consider" items).

### Step 1: Classify each comment, then tabulate — BEFORE any edits

For every comment, decide which of three buckets it falls into:

- **Actionable** — a clear, unambiguous fix where you're confident in the right approach (e.g. "rename this var", "this branch is dead, drop it", "missing type annotation", "extract this constant"). Bot suggestions like refactoring repetitive code default to actionable.
- **Question** — the comment asks for design input or weighs trade-offs. Trigger phrases include: "should we…", "would it be overkill to…", "is this worth it…", "what do you think…", "I'm wondering if…", "couldn't this be…", "do we need…". These are NOT directives — they're invitations to discuss. **Treat anything that ends with a question mark as a Question by default.**
- **Affirmation** — pure FYI/agreement/opinion with no code change implied.

Build the table BEFORE editing anything. Include a **Type** column:

| # | Source | Type | Item | Status |
|---|--------|------|------|--------|
| 1 | @user | Question | "Should we use `examples=` instead of description?" | _pending discussion_ |
| 2 | deep-review | Actionable | Drop dead `Foo \| None` branch in `_is_foo_field` | _pending fix_ |
| 3 | @user | Affirmation | This model will be reused by the next feature | _no action_ |

- **Source**: who left the comment (e.g. `@username`, `deep-review`, `Vercel`)
- **Final status values** (after Step 3):
  - For Actionable rows: `Fixed`, `Skipped — <reason>`, or `Flagged — <reason>`
  - For Question rows: `Resolved — <agreed direction>`
  - For Affirmation rows: `No action`

### Step 2: If the table has any Questions, discuss them first — DO NOT start editing yet

This is the most common failure mode of this skill: jumping to implementation when the user actually wanted to discuss. Don't.

For every Question item:

- State your interpretation of what's being asked
- Give your recommendation with one or two sentences of reasoning (the *why*, not just the *what*)
- Lay out the realistic alternatives so the user can pick

Group **all** discussion points into one response. Use `AskUserQuestion` when the choices are clean and mutually exclusive; use prose when the questions are open-ended or interrelated. Don't drip-feed questions across multiple turns.

Wait for the user's direction before any edits. Once answered, update each Question row's `Status` to reflect the agreed direction, then continue.

### Step 3: Implement

Work through Actionable items + the now-resolved Questions. For each: read the relevant code, make the fix, move on. Flag anything that turns out to be ambiguous mid-implementation back to the user instead of guessing.

After all fixes, update the table with final statuses (per the vocabulary above), run `/lint`, commit, push, and capture this commit's SHA with `git rev-parse --short HEAD`. Step 4 references it.

## Step 4: Post the summary on the PR

Once the feedback commit is pushed, post **one** top-level PR comment summarizing what was addressed. The user reviews the PR on GitHub before clicking Merge in the UI, so this comment is the canonical record of what `/address-pr-feedback` did.

**Posting mechanics**: write the markdown body to a tmpfile and pass it via `--body-file`, not `--body "..."`. The body contains backticks, pipes, code fences, and quoted user-comment snippets that may themselves contain backticks or `$` — shell-quoting that as a single argument is fragile and silently mangles rows. Either:

```bash
gh pr comment <number> --repo <owner>/<repo> --body-file /tmp/address-summary.md
```

or via stdin: `... --body-file -` and pipe the markdown in.

The comment posts under the user's GitHub account (the `gh` CLI is authenticated as the user, not as `claude[bot]`) — the heading below makes the auto-generated nature explicit.

Shape:

```markdown
### Address summary

_Auto-generated by `/address-pr-feedback`._

**Feedback addressed** (commit `<sha>`):

| # | Source | Type | Item | Status |
|---|--------|------|------|--------|
| 1 | @user | Question | "Should we use `examples=` instead of description?" | Resolved — agreed, used `examples=` |
| 2 | deep-review | Actionable | Drop dead `Foo \| None` branch | Fixed |
| 3 | @user | Affirmation | This model will be reused by the next feature | No action |
```

Rules:
- One comment. Don't split the table across several.
- The example table above shows all three row types (Question/Actionable/Affirmation) for illustration; drop the row types that didn't appear in this PR's actual feedback.
- Substitute the SHA captured at the end of Step 3 into the template.

## Summary

In the chat, present the **same content** that was posted to the PR comment — the full feedback status table. Do not abbreviate. Then link to the PR comment URL so the user can open it on GitHub if they want.
