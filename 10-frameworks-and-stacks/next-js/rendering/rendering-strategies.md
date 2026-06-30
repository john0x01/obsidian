# Rendering Strategies

Every rendering strategy is an answer to a single question: **when is the HTML for this response produced — at build time, at request time, or somewhere in between?** Next.js 15's App Router doesn't make you pick one strategy for the whole app; it derives the strategy per *route segment* from what your code actually does, which is the mental shift that trips up engineers coming from the Pages Router's explicit `getStaticProps`/`getServerSideProps` dichotomy.

## The Spectrum Of When HTML Is Produced

Think of a single time axis. Strategies differ only in where on it the HTML is computed:

```
build ───────────────── revalidate ───────────────── request
  │                          │                            │
 SSG                ISR (time / on-demand)               SSR
 (prerendered once)  (prerendered, refreshed later)   (computed per request)
```

- **CSR** — server ships an empty/skeletal shell; the browser fetches data and renders. In the App Router this is *not* a route-level mode; it's what a `'use client'` subtree does after hydration. There is no "CSR page" — there is server-rendered HTML plus client islands.
- **SSG (static)** — HTML produced once at `next build`, served from the Full Route Cache / CDN. Cheapest, fastest TTFB, but frozen until rebuild.
- **ISR** — static HTML that is *re-generated* on a schedule (`revalidate = N`) or by an explicit invalidation call (`revalidatePath`/`revalidateTag`). Build-time freshness with request-time recency.
- **SSR (dynamic)** — HTML computed on every request. Required when output depends on the request (cookies, headers, search params) or on uncacheable data.
- **Streaming** — orthogonal to the above: the *delivery* of that HTML in chunks rather than one blob, gated by Suspense.
- **PPR** — combines static and dynamic *in one route*: a prerendered shell with dynamic holes streamed in.

The crucial insight: SSG/ISR/SSR are points on the *when* axis; streaming and PPR are about *how the response is delivered*. They compose.

## How The App Router Decides (Static By Default)

There is no `export const mode = 'ssr'`. A segment is **static unless something forces it dynamic**. Next 15 performs a build-time prerender pass; if it succeeds for a segment without hitting a dynamic dependency, that segment is static. A route becomes dynamic when it reads request-time data or opts out of caching:

- Reading `cookies()`, `headers()`, `draftMode()`, or the `searchParams` prop.
- A `fetch` marked `cache: 'no-store'`, or any uncached data access.
- `export const dynamic = 'force-dynamic'` (or `force-static` to go the other way).
- `export const fetchCache`, `revalidate`, `dynamicParams` route-segment config.

This is why a seemingly innocent `const c = cookies()` in a deep layout can flip an entire route to dynamic — dynamism propagates *up* the tree to the nearest static boundary. The dynamic APIs in Next 15 are async (`await cookies()`), reinforcing that touching them is a request-time act.

```ts
// Forces a segment dynamic — every request recomputes:
export const dynamic = 'force-dynamic'
// Or, more honestly, just read a request input:
const token = (await cookies()).get('session')
```

`generateStaticParams` is how a dynamic route (`[slug]`) re-enters the static world: you enumerate params at build time so each becomes an SSG page; `dynamicParams` controls whether unlisted params render on-demand (ISR-like) or 404.

## How To Choose

Decide per segment by asking three questions in order:

1. **Does the output depend on the individual request?** (auth, geo, A/B, per-user) → dynamic/SSR. Don't fight it.
2. **If not, how stale can it be?** Never stale → SSG (rebuild to update). Tolerable staleness → ISR with `revalidate`, or on-demand invalidation if you control the write path. The on-demand path (revalidate on publish) gives static performance with near-real-time correctness and is the most underused sweet spot.
3. **Is the dynamic part a small slice of an otherwise static page?** → don't make the whole route dynamic; isolate the dynamic part behind Suspense (streaming) or, ideally, PPR.

## Senior Pitfalls And Philosophy

- **"Dynamic" is a failure of caching, not a feature.** The expensive default in the old world (SSR everything) is now the *exception*. Treat every dynamic route as a deliberate decision with a justification.
- **Per-request ≠ per-route.** The biggest mindset error is choosing one strategy globally. The whole point of the App Router is granularity: a static marketing shell, an ISR product grid, and a dynamic cart can coexist in one render tree.
- **Streaming doesn't make a page dynamic.** Static pages can stream too; loading UI and Suspense are about *delivery*, not freshness.
- **The dev server lies.** Local dev is always dynamic-feeling; you must read `next build` output (the ○ ● ƒ markers) to know each route's real strategy. Verify, don't assume.

The philosophy: collapse the old binary (`getStaticProps` vs `getServerSideProps`) into a single continuum and let the *data dependencies* of each segment pick a point on it — defaulting to the cheapest one (static) and degrading to dynamic only where the data demands it.

## See Also
- [[static-and-isr]]
- [[streaming-ssr]]
- [[partial-prerendering]]
- [[hydration]]
- [[10-frameworks-and-stacks/react/philosophy/composition-model|React Composition Model]]
