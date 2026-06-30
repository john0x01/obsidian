# App Router File Conventions

The App Router is configured almost entirely by *filenames*. A segment folder becomes routable and renderable purely by the special files you drop into it. The deep idea: each convention maps to a specific position in the rendered React tree and a specific lifecycle. Knowing *which file becomes which wrapper, and in what order they nest*, is what lets you reason about why a layout persisted, why an error was caught here and not there, or why a loading state appeared.

## The Files and What Each Becomes

- **`layout.tsx`** — Shared UI that wraps the segment and all descendants. Receives `children`. **Persists across navigation** within its subtree (does not re-mount, preserving state). The root `layout` is mandatory and owns `<html>` and `<body>`.
- **`page.tsx`** — The leaf UI for a route. A segment is only publicly accessible if it (or a descendant) has a `page` or `route`. Receives `params` and `searchParams` (both **Promises** in Next 15 — you `await` them).
- **`loading.tsx`** — Sugar for a React `<Suspense>` boundary wrapping the segment's `page`. Shown instantly while the server streams the segment's content. Enables the "instant shell" UX.
- **`error.tsx`** — A React Error Boundary wrapping the segment (but **not** its own layout). Must be a Client Component (`'use client'`); receives `error` and a `reset()` function. Catches errors thrown in the page and nested children below it.
- **`not-found.tsx`** — UI rendered when `notFound()` is invoked in the segment, or for unmatched routes. The root `not-found` also handles global 404s.
- **`template.tsx`** — Like `layout`, but **re-mounts on every navigation**, giving each child a fresh instance (new state, re-run effects, replayed enter animations). Use when you explicitly want *non*-persistence.
- **`route.ts`** — Defines an HTTP endpoint instead of UI. A segment **cannot have both `page` and `route`** at the same path — they're mutually exclusive. See [[route-handlers]].
- **`default.tsx`** — Fallback UI for an unmatched **parallel route** slot during navigation. See [[advanced-routing]].
- **`global-error.tsx`** — Top-level error boundary that replaces the root layout (it must render its own `<html>`/`<body>`), catching errors in the root layout itself.

## How They Compose the Tree

The wrappers nest in a fixed order around the page. Conceptually, for a single segment with all files present, the framework produces:

```
<Layout>
  <Template>
    <ErrorBoundary fallback={error.tsx}>
      <Suspense fallback={loading.tsx}>
        <NotFoundBoundary fallback={not-found.tsx}>
          <Page />
        </NotFoundBoundary>
      </Suspense>
    </ErrorBoundary>
  </Template>
</Layout>
```

Two consequences fall directly out of this ordering:

- **`error.tsx` sits *inside* its sibling `layout.tsx`.** So an error in the page is caught and the layout stays visible — but an error *thrown by the layout itself* is **not** caught by that segment's error boundary; it bubbles to the parent segment's `error.tsx`. This is the single most common confusion.
- **`loading.tsx` is just a Suspense fallback.** It appears while the server hasn't finished streaming. Anything that suspends below it (an `async` Server Component awaiting data) triggers it.

Across segments these structures *stack*: the parent layout wraps the child layout wraps the child page, each contributing its own loading/error boundaries at its own depth.

## Route Segment Config

Each `layout`, `page`, or `route` file can export config constants that tune rendering and caching for that segment:

- `export const dynamic = 'auto' | 'force-dynamic' | 'error' | 'force-static'` — force dynamic or static rendering, or error if dynamic APIs are used.
- `export const revalidate = false | 0 | number` — segment-level ISR revalidation window in seconds.
- `export const fetchCache`, `runtime` (`'nodejs' | 'edge'`), `preferredRegion`, `dynamicParams`, `maxDuration`.

These are read at build/compile time, so they must be statically analyzable literals — not computed at runtime.

## The Metadata API

Metadata is data, not JSX. Two forms:

- **Static:** `export const metadata: Metadata = { title, description, openGraph, ... }`.
- **Dynamic:** `export async function generateMetadata({ params }) { ... }` — can `await` data and is deduped against your page's own fetches.

Metadata merges down the tree: child segments override or extend parent metadata (e.g., `title.template` in a layout fills in around a child's `title.default`). File conventions like `icon.png`, `opengraph-image.tsx`, `sitemap.ts`, and `robots.ts` are also recognized and generate the corresponding `<head>` tags or routes automatically. Next streams `<head>` metadata as part of the RSC payload, so dynamic metadata doesn't block the document shell.

## Senior Pitfalls

- Putting per-child data fetching in `layout.tsx` — it won't re-run on sub-navigation.
- Expecting `error.tsx` to catch its own layout's errors (it can't).
- Forgetting `params`/`searchParams` are Promises in Next 15 (a frequent migration break).
- Using `template.tsx` "just in case" — it discards the persistence win that makes layouts valuable.

## See Also

- [[app-router-model]]
- [[server-and-client-components]]
- [[route-handlers]]
- [[advanced-routing]]
