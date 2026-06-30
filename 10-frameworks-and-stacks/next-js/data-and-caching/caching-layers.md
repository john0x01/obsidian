# The Four Caching Layers

Next.js (App Router) maintains four distinct caches, and almost all caching confusion comes from treating them as one thing. They differ in *what* they store, *where* they live, and *how long* they last. Two are server-side and concern data; one is server-side and concerns rendered output; one is client-side and concerns navigation. Knowing which layer is responsible for a stale screen is the entire skill.

## The layers

- **Request Memoization** — *Scope:* a single server render pass for one request. *Stores:* the in-flight/resolved result of identical `fetch()` calls (same URL + options) and React `cache()`-wrapped functions. *Lifetime:* dies when the render finishes; it is not persisted. Purpose: lets you co-locate the same fetch in many components without N duplicate requests. It is React's deduplication, not really "caching" in the persistence sense.

- **Data Cache** — *Scope:* server-wide, across requests *and* deployments (persisted; on Vercel it's a managed store). *Stores:* the *results of `fetch()`* (and `unstable_cache` outputs). *Lifetime:* until revalidated by time (`next.revalidate`) or tag/path invalidation. This is the durable HTTP-data cache. In Next 15 a bare `fetch` is **not** stored here by default (see below).

- **Full Route Cache** — *Scope:* server-wide, per route, established at **build time** for static routes. *Stores:* the rendered RSC payload **and** the HTML for a route. *Lifetime:* persists until the route is revalidated or redeployed. Only **static** (prerendered) routes populate it; dynamic routes are rendered per-request and skip it. It sits *downstream* of the Data Cache: a static route's rendered output is cached, and the data that produced it is separately cached in the Data Cache.

- **Router Cache (client-side)** — *Scope:* a single user's browser, in memory for the session. *Stores:* the **RSC payload** of visited and prefetched route segments. *Lifetime:* in Next 15, the default `staleTime` for **page** segments is **0** (pages are refetched on navigation), while prefetched/layout data has a short window; it's cleared on full reload and can be invalidated via `router.refresh()`, server actions, or `revalidatePath`/`revalidateTag` propagating to it. See [[router-internals]].

## How they interact

```
                    ┌─────────────────────── SERVER ───────────────────────┐
  request ─▶ render ─▶ fetch(url) ─▶ [Request Memoization]  (this render only)
                                          │ miss
                                          ▼
                                     [Data Cache]  (cross-request, persisted)
                                          │ miss
                                          ▼
                                     origin / DB / API
                          │
        rendered RSC+HTML │ (static routes only, at build/revalidate)
                          ▼
                    [Full Route Cache]  (server, per route)
                          │  RSC payload sent on navigation
   ┌────────── CLIENT ────┼────────────────────────────────────────────────┐
   │                      ▼                                                  │
   │                [Router Cache]  (in-memory, per session)                │
   └───────────────────────────────────────────────────────────────────────┘
```

Read it top-down for a single request: memoization collapses duplicate fetches *within* the render; survivors fall through to the Data Cache (cross-request); the rendered result of a static route is captured by the Full Route Cache; the client's Router Cache holds RSC payloads to make navigation instant. A revalidation event (time or tag) invalidates the Data Cache entry, which in turn forces the Full Route Cache entry to be regenerated on the next request, and signals the Router Cache to drop its copy.

## Addressing the common confusion

The misconception is "I called `revalidatePath` but the page is still stale" or "I set `revalidate` but my client doesn't update." These are almost always a *layer mismatch*:

- **Stale after a fetch revalidation but fresh on hard reload** → the **Router Cache** is serving an old RSC payload on the client. The server data is fine; the browser is showing cached navigation. Fix with `router.refresh()` or ensure the mutation path calls `revalidatePath`/`revalidateTag`, which also marks the client cache.
- **Data Cache vs Full Route Cache** are frequently merged in people's heads. The Data Cache stores *fetch results*; the Full Route Cache stores *rendered output*. A route can be dynamic (no Full Route Cache) yet still hit the Data Cache for individual fetches, and vice versa: a statically rendered route's fetches might be uncached but the whole rendered route is cached at build.
- **Memoization is not the Data Cache.** Memoization is why the same fetch in five components hits the network once *this render*; it explains nothing about cross-request freshness. Don't reach for `revalidateTag` to "fix" duplicate requests — that's memoization's job and it's automatic.

## Senior notes

- The Full Route Cache and Data Cache are *decoupled but linked*: invalidating data forces route re-render, but a dynamic route never populated the route cache to begin with.
- The Router Cache is the layer you have least direct control over and it causes the most "ghost staleness." Treat any mutation as needing an explicit invalidation signal to the client, not just the server.
- These caches are an opt-out-able stack, not a single dial. Diagnosing staleness means asking *which* layer holds the stale copy. See [[revalidation]] for the levers.

## See Also

- [[revalidation]]
- [[data-fetching]]
- [[router-internals]]
- [[server-actions]]
- [[06-software-design/system-design/caching|Caching]]
- [[11-applied/web-platform/http-caching|HTTP Caching]]
