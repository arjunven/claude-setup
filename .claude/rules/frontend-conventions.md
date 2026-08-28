---
paths:
  - "frontend/**/*.ts"
  - "frontend/**/*.tsx"
  - "frontend/**/*.js"
  - "frontend/**/*.jsx"
description: Frontend code conventions
---

## Next.js

- **Read bundled docs before writing Next.js code — or before asserting anything about a Next.js file convention or API.** Version-matched docs live in `frontend/node_modules/next/dist/docs/`. Always consult them instead of relying on training data for routing, data fetching, or any Next.js API. Diagnosis counts: a claim about how a convention works is exactly as wrong as code built on it, and lands in tickets and plans where it outlives the conversation.

- **A negative filesystem check for a convention filename proves nothing.** `ls middleware.ts` returning empty says only that the *name you guessed* is absent, and the name is precisely what stale training data gets wrong — so the check is circular: it takes the assumption as input and returns it as a conclusion. Next 16 renamed `middleware.ts` to `proxy.ts`; searching for the old name produced "this app has no middleware layer" while `src/proxy.ts` sat there the whole time. Grep the bundled docs for the concept (`find node_modules/next/dist/docs -iname "*middleware*"`) before concluding the repo doesn't use it. Generalizes past Next — the same trap applies to any renamed API.

- **Don't run `bun run build` locally to verify a change — let Vercel CI build it.** A local production build is not the build that ships and fails for reasons that have nothing to do with the diff (env vars Vercel injects that the local `.env` does not carry, framework-internal routes that prerender differently locally). For local signal use the type-check and lint that `/lint` already runs (`frontend-type-check` is the same `tsc` CI runs); push and read the Vercel check for the build itself.

- **Rendering mode and backend coupling** — a page that fetches from the backend at the top level (a server component `await`ing our API) is **statically prerendered at build by default**, so `next build` then depends on the backend being reachable. The trigger for "build vs request time" is invisible: a route is dynamic (fetches per request) only if it reads request data on the server — `searchParams`/`cookies()`/`headers()`, or a feature-flag SDK that reads them internally. Reading search params **client-side** via nuqs/`useSearchParams` does NOT make the page dynamic; it only triggers a CSR bail-out scoped by `<Suspense>`. Consequences:
  - If a page must not couple the build to the backend, either it is already dynamic (reads `searchParams`/a flag on the server) or make it dynamic deliberately, or don't fetch from the backend at build.
  - State the rendering mode at the fetch / `generateMetadata` site so it isn't silently flipped by an unrelated cleanup (e.g. removing a `searchParams` read).

- **A route-level `loading.tsx` flushes the shell before metadata resolves — never add one to a route whose `<head>` or redirects matter.** It wraps the page in a Suspense boundary *above* the page component, so Next sends the opening HTML immediately. Two consequences, both invisible in code review and visible only on a deployed URL:
  - `generateMetadata` output is appended to `<body>` instead of `<head>`. Titles survive (Google reads them from the rendered DOM) but `rel="canonical"` outside `<head>` is ignored.
  - `redirect()` / `permanentRedirect()` cannot set a status once the response has flushed, so they degrade to a client-side `<meta http-equiv="refresh">` on a **200**. Every URL the route meant to redirect stays a live, indexable duplicate.

  The trade runs both ways, so decide per route: a **dynamic** route is not prefetched at all without `loading.js`. Indexable content routes should skip it; app-shell and tool routes that nobody canonicalizes are free to keep it.

- **Verify `<head>` placement, canonical, and redirect status against a deployed preview — the code cannot tell you.** Whether a tag lands in `<head>` and whether a redirect is a 3xx or a meta refresh are both emergent from streaming, so reading the route proves nothing. Fetch the URL, assert the status line first (a firewall 429/403 is not app behavior), then assert the tag's byte offset against `</head>` and the redirect's status. Sample a few times: a placement that flips between identical requests is a race, not a pass.

- **noindex vs canonical on parameterized tool URLs** — for a tool page that encodes state in the URL (`?tool=…`), keep only the bare landing in the search index. noindex the parameterized states, and do **not** also point their `canonical` at the bare URL — `noindex` + a cross-`canonical` are conflicting signals. Subtlety: returning an unset `alternates`/`openGraph` does NOT drop the canonical/og:url — Next.js merges metadata shallowly, so the route inherits the **root layout's** homepage canonical; use `canonical: null` to actually suppress it. Don't hand-roll this per page: own the policy in one helper (`lib/seo.ts`) and have each page call it.

# Frontend Conventions

## Component Style

- **`const` arrow functions** — `const ComponentName = () => {}`, not function declarations
- **Typed props, not `React.FC`** — use `const Component = ({ foo }: Props) => {}`, not `const Component: React.FC<Props> = ({ foo }) => {}`. `React.FC` is legacy. When touching a file that uses `React.FC`, migrate it to plain typed props and remove the `import type React from "react"` if no longer needed.
- **`handle` prefix** for event handlers — `handleClick`, `handleSubmit`
- **kebab-case files** for components — `alternates-table.tsx`. When modifying a component file that still uses PascalCase, rename it to kebab-case as part of the change. Keep the React component name PascalCase — only the filename and its importers change.
- **No IIFEs in JSX** — extract to a component or `useMemo` variable instead
- **No mutation in pure functions** — spread first, then override
- **Use shadcn/ui components** from `frontend/src/components/ui/` — always check for an existing primitive before building custom UI. Use `<Button>` not `<button>`, `<Input>` not `<input>`, shadcn `Pagination` not a custom pagination component, etc. If a shadcn component exists for the pattern, compose with it rather than reimplementing. If no matching component exists locally, use the `/shadcn` skill to search the registry for an applicable component to install before building from scratch.
- **Absolute imports for project source** — use `@/lib/foo`, `@/components/foo`, `@/hooks/foo`, etc. Never use relative paths (`./foo`, `../foo`) for files outside the current directory. Same-folder relative imports are tolerated but absolute is preferred. External-package imports stay bare. Biome's import organizer will sort them; the rule is about which form to write, not the order.

## State Typing

- **Use generated backend types for state** — when storing data from the API (configs, filter state), use the generated type from `@/client`, not generic `Record<string, ...>`. This preserves type safety through the whole component lifecycle and avoids double-casting when sending data back.
- **Spread to modify, don't destructure into Records** — to update a field on a typed object, use `{ ...original, [key]: newValue }` (preserves the type) not `Object.entries()` → rebuild into a `Record` (loses the type).

## Naming

- **One constant for "no value"** — export a `NO_VALUE` constant from a shared utils module and use it everywhere. Never use literal `"-"`, `"—"`, `"—"`, or `"N/A"` for placeholder text in tables.
- **Normalization helpers have one home** — any identifier that gets cleaned before use as a cache key, dedup key, or lookup (`.trim().toUpperCase()` on an ID, say) goes through a single function that mirrors the backend's normalizer. Never inline it.
- **Symbols and units come from one module** — never spell a unit or symbol inline in a component, and don't declare a local per-file constant either; that is still a second home for the string. Add it to the shared module and import it. Migrate touch-as-we-go.

## URL State Sync

- **Never sync URL on field change** — `onChange` handlers update local state only. URL is synced **after** the search response succeeds, derived from the response data (not local state). This prevents sharing URLs with stale or intermediate values.
- **Wait for initial data before mount-time search** — the URL-sync effect must depend on the initial data being loaded. Firing before data is ready sends incomplete requests and the response overwrites the URL params the user pasted in.
- **Use a `hasSyncedFromUrlRef` to prevent loops** — set it to `true` when writing URL state from a response; check it in the URL-sync effect to avoid re-triggering search. Don't use a `shouldUpdateUrl` flag to skip URL writes on mount — always write URL from the response, and rely on the ref to break the cycle.

## Styling

- **Never native browser tooltips** — no `title` attributes for hover text. Use the shadcn `Tooltip` / `TooltipTrigger` / `TooltipContent` from `@/components/ui/tooltip`: instant, styled, theme-aware. Native `title` has a ~1 s delay and unstyled rendering that reads as broken.
- **Tailwind classes only** — no custom CSS
- **`cn()` utility** from `frontend/src/lib/utils.ts` for conditional classes
- **Rendered markdown goes through one typography wrapper** (we use shadcn/typeset) with one shared `markdownComponents` map spread into every `ReactMarkdown`; add per-renderer `components` overrides only for elements that need app-specific **behavior**, never for pure styling.

## Dev toolbar

- **A dev-only banner (`features/dev-toolbar/`) is the home for development-only rendering controls.** When UI work hits a fork in the road — two plausible renderings, a debug overlay, data-point markers — put a toggle in the dev toolbar and evaluate on real data, rather than deciding blind or wiring up a hidden query param. Toggle state is inert in production builds: reads return the defaults, writes no-op.
- **Two tiers of toggle — label them.** *Permanent* toggles are standing dev kit. *Experiment* toggles exist to evaluate a UI fork on one branch and are torn out once the decision lands — prefix them `exp:` in the toolbar UI and mark the code with `experiment(<ticket>)` so teardown is greppable. Don't let experiment toggles accumulate across branches.

## Organization

### File layout — feature-folder co-location

The target shape for `frontend/src/`:

- `app/` — Next.js routing, layouts, pages (framework-mandated).
- `features/<feature>/` — feature-owned code: types, raw modules, hooks, contexts, components, and tests all in one folder. **New domain code goes here**, not in top-level `hooks/`, `contexts/`, or a multi-feature `lib/` file.
- `components/ui/` — shadcn/ui primitives plus shared custom components. Vocabulary used across features.
- `lib/` — genuinely cross-cutting utilities (`utils.ts` with `cn()`, etc.). If a utility is only used by one feature, it lives in that feature's folder.
- `client/` — generated openapi-fetch code. Don't restructure.
- `hooks/`, `contexts/` — legacy top-level folders. **New hooks/contexts go in their feature folder**, not here.

A typical feature folder:

```
features/
  search-quota/
    search-quota.ts            (raw module + helpers + types)
    search-quota-context.tsx   (provider)
    use-user-tier.ts           (hook)
    search-quota.test.ts       (test, co-located)
    components/                (only if the feature has its own UI)
```

- **Migrate as you touch — any size of change.** When you modify any file currently in `hooks/`, `contexts/`, a multi-feature `lib/` location, or `components/` without a feature subfolder, move it to its feature folder as part of the same PR — even for one-line edits. The codebase migrates by attrition, and every touch shifts it closer to the target shape. Use `git mv` so history follows the file; update importers in the same commit.
- **Use `ui/table` for all tables** — `<Table>`, `<TableHeader>`, `<TableRow>`, `<TableHead>`, `<TableBody>`, `<TableCell>` from `@/components/ui/table`, not raw `<table>`/`<tr>`/`<td>`. For interactive tables with sorting/filtering, use `ui/data-table/` which wraps TanStack Table with the same shadcn primitives.

### Tests

- **Co-locate unit tests with the file under test** — `foo.test.ts` next to `foo.ts` in the same folder. Don't add to `src/__tests__/`. Bun's test runner finds tests at any path; the parallel `__tests__/` tree is a legacy Jest convention with no benefit here.
- **When touching code that has a test in `__tests__/`, move the test next to its source** as part of the change. Migrate-on-touch, same as the file-layout rule above.
- **E2E tests** (Playwright) live in a top-level `frontend/e2e/` folder, not under `features/`. They exercise multiple modules end-to-end and don't co-locate to a single source file.
