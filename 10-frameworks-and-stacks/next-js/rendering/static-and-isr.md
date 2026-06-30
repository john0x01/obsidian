# Static Rendering And ISR

Static rendering is the App Router's default: produce a route's HTML (and its RSC payload) **once, at `next build`**, and serve it from cache to every visitor. Incremental Static Regeneration (ISR) keeps that static-serving performance while letting the prebuilt output be *refreshed* over time — either on a clock (time-based) or by an explicit invalidation (on-demand). Understanding ISR well means understanding three caches and one HTTP pattern that sit underneath it.

## What "Static" Actually Produces

At build, Next runs a prerender pass per static segment and emits two artifacts: the **HTML** (for the initial document / SEO / no-JS) and the **RSC payload** (the serialized React Server Component tree the client uses for navigations). Both are stored in the **Full Route Cache** on the server build output. A static route therefore costs essentially zero compute at request time — it's a file read, often elevated to the CDN edge.

A route stays static only if nothing in its tree reads request-time input (`cookies()`, `headers()`, `searchParams`) or opts out of data caching. The moment it does, that segment becomes dynamic and leaves the Full Route Cache.

## generateStaticParams

For dynamic segments (`app/blog/[slug]/page.tsx`), `generateStaticParams` enumerates which parameter values to prerender at build time, turning one dynamic route into N static pages:

```ts
export async function generateStaticParams() {
  const posts = await getPosts()
  return posts.map((p) => ({ slug: p.slug }))
}
// Control unlisted params:
export const dynamicParams = true  // default: render on-demand, then cache (ISR-like)
// dynamicParams = false → unlisted slugs 404
```

With `dynamicParams = true`, a request for a slug not built at compile time is rendered on first hit and then cached — this is ISR's "render on demand and persist" behavior even without a `revalidate` value.

## Incremental Static Regeneration

ISR layers a refresh policy on top of static output.

**Time-based.** Set `revalidate` (seconds) via segment config or per-`fetch`:

```ts
export const revalidate = 3600        // whole segment
// or, granularly:
await fetch(url, { next: { revalidate: 3600 } })
```

**On-demand.** Invalidate explicitly from a Server Action or Route Handler when the underlying data changes — the better pattern when you control the write path, because it gives static perf with near-real-time correctness:

```ts
import { revalidatePath, revalidateTag } from 'next/cache'
revalidateTag('products')        // every fetch tagged 'products' is now stale
revalidatePath('/blog/[slug]')   // a specific route
```

Tags are attached at fetch time (`fetch(url, { next: { tags: ['products'] } })`), forming a dependency graph: one `revalidateTag` call fans out to every cached entry carrying that tag, across routes. This is the mechanism that makes "revalidate on publish" trivial.

## Stale-While-Revalidate: The Core Semantics

ISR does **not** make a user wait for regeneration. It implements stale-while-revalidate:

```
request after revalidate window elapsed:
  1. serve the STALE cached page immediately  (fast)
  2. trigger a background regeneration
  3. on success, atomically swap the cache entry
  4. the NEXT visitor gets the fresh page
```

So the visitor who "triggers" revalidation never pays for it — they get stale HTML; a later visitor gets fresh HTML. This is why ISR feels free and why a low-traffic page can stay stale far longer than its `revalidate` value: regeneration only happens when a request arrives *after* the window. There is no cron sweeping your pages.

## The Three Caches You Must Keep Straight

1. **Request Memoization** — dedupes identical `fetch` calls within a *single* render pass. Per-request, in-memory, automatic.
2. **Data Cache** — persists `fetch` results across requests and deployments; governed by `revalidate`/`tags`. This is what `revalidateTag` invalidates.
3. **Full Route Cache** — the prebuilt HTML + RSC payload for a static route. Invalidated when its underlying Data Cache entries are revalidated, or by `revalidatePath`.

`revalidateTag` invalidates the Data Cache, which cascades to drop the affected Full Route Cache entries. Getting these confused is the #1 source of "why is my page still stale / why won't it cache" confusion.

## Senior Pitfalls And Philosophy

- **`revalidate` is a *maximum staleness*, not a *refresh interval*.** No traffic, no regeneration. Don't model it as a cron.
- **The first visitor after the window sees stale content by design.** If you need the triggering user to see fresh data (e.g., after their own mutation), use on-demand `revalidatePath`/`revalidateTag` inside the Server Action, not time-based ISR.
- **One dynamic API poisons the whole route's staticness.** A `headers()` read in a shared layout silently demotes every child route from the Full Route Cache. Audit `next build` output for the static/dynamic markers; the dev server won't show you this.
- **Tag everything you might invalidate.** Tags cost nothing at write time and are the only ergonomic way to do surgical, cross-route invalidation later.
- **Build-time fetch failures.** If `generateStaticParams` or a static fetch throws at build, you get a build failure or a fallback to runtime rendering — decide deliberately rather than discovering it in CI.

Philosophy: treat freshness as a *budget* you spend deliberately. Static is the free baseline; ISR lets you buy exactly as much recency as the data demands — by the clock when you don't control writes, on-demand when you do — without ever giving up cache-served performance.

## See Also
- [[rendering-strategies]]
- [[streaming-ssr]]
- [[partial-prerendering]]
- [[hydration]]
