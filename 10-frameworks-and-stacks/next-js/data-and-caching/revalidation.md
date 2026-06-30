# Revalidation And Cache Control

Revalidation is how you tell the server-side caches (the Data Cache and, downstream, the Full Route Cache) that stored data is no longer fresh. There are two mechanisms — **time-based** and **on-demand** — plus the ways you opt *out* of caching entirely. Next 15 changed the defaults significantly, so reasoning about freshness now starts from "uncached unless I asked for it."

## The Next 15 default shift

In Next 13/14, a bare `fetch()` in a Server Component was cached in the Data Cache by default (`force-cache`). In **Next 15 the default is uncached** — `fetch` behaves like `cache: 'no-store'` unless you explicitly opt in. Routes are likewise no longer aggressively cached by default in the same way; `GET` Route Handlers and client router caching defaults were also relaxed. The practical consequence: caching is now a deliberate act. If you want a fetch in the Data Cache, you say so:

```ts
// opt INTO caching, revalidate every 60s
fetch(url, { next: { revalidate: 60 } });
// opt into indefinite caching
fetch(url, { cache: 'force-cache' });
```

This inverts the old mental burden. Previously you spent effort *opting out* of stale caches; now you spend effort *opting in* where caching actually helps. It is safer-by-default (fewer surprise-stale screens) at the cost of more explicit configuration and, if you forget, more origin hits.

## Time-based revalidation

`next: { revalidate: N }` (seconds) marks a cached fetch as stale after N seconds. The model is **stale-while-revalidate**: the first request *after* the window still serves the stale entry, triggers a background regeneration, and subsequent requests get the fresh value. You never block a user on the refresh. A route-level `export const revalidate = N` applies the same to the route's Full Route Cache. `revalidate = 0` makes the route dynamic. `revalidate: false` (or `Infinity`-equivalent via `force-cache`) caches indefinitely.

The key nuance: time-based revalidation is *lazy and traffic-driven*. Nothing regenerates on a timer in the background; it regenerates on the next request after expiry. A zero-traffic route stays stale forever until someone visits.

## On-demand revalidation

When you know exactly *when* data changed (after a mutation), don't wait for a timer — invalidate precisely:

- `revalidatePath('/blog/[slug]', 'page')` — purges the Data Cache entries and route cache for a path.
- `revalidateTag('posts')` — purges every cached fetch tagged `posts`.

**Cache tags** are the better primitive for anything non-trivial. You tag a fetch at read time and invalidate by tag at write time, decoupling the writer from the readers' URLs:

```ts
// reader (anywhere)
fetch('https://api/posts', { next: { tags: ['posts'] } });
// writer (a Server Action / Route Handler)
revalidateTag('posts'); // every fetch tagged 'posts' is now stale
```

Both functions run server-side (in Server Actions or Route Handlers) and also mark the client **Router Cache** as needing refresh, which is why a mutation followed by `revalidateTag` usually updates the UI without a manual `router.refresh()`. For non-`fetch` sources, wrap the accessor in `unstable_cache(fn, keyParts, { tags, revalidate })` to make ORM/SDK reads taggable and time-revalidatable.

## Opting out / forcing dynamic

Several signals force a route or fetch to be dynamic (uncached, rendered per request):

- `cache: 'no-store'` on a fetch (now the default in 15).
- Reading **dynamic APIs**: `cookies()`, `headers()`, `searchParams`, `draftMode()`. Touching any of these opts the route out of static rendering. In Next 15 several of these (`cookies`, `headers`, `params`, `searchParams`) became **async** and must be awaited.
- `export const dynamic = 'force-dynamic'` or `export const fetchCache = 'force-no-store'` at the route level.
- `export const revalidate = 0`.

Use `connection()` (Next 15) when you explicitly want to defer to request-time even without reading a dynamic API.

## Mental model

Hold three questions in mind, in order:

1. **Is this fetch even cached?** (In 15, no, unless you opted in.) If uncached, revalidation is irrelevant — it's always fresh and always hits origin.
2. **If cached, what expires it?** Time (`revalidate`) is a *best-effort staleness window*; on-demand (`tag`/`path`) is *exact invalidation*. Prefer tags for correctness, time for cheap freshness on data with no clear change event.
3. **Did the invalidation reach the client?** Server revalidation updates the Data/Route caches; the **Router Cache** is a separate client layer (see [[caching-layers]]). On-demand functions signal it; a pure time-based expiry plus a soft client navigation can still show a stale RSC payload until refresh.

## Pitfalls

- **Treating `revalidate` as a cron.** It's lazy SWR, not a scheduler. Low-traffic routes stay stale.
- **Forgetting tags are read-side.** You must tag at the fetch; `revalidateTag` does nothing for fetches that never declared the tag.
- **Assuming the client updated.** Server-side freshness ≠ client-side freshness; verify the Router Cache got the signal.
- **Over-caching dynamic data in 15 by reflex from 14 habits**, or the inverse — forgetting to opt *in* and quietly hammering an upstream API on every request.

## See Also

- [[caching-layers]]
- [[server-actions]]
- [[data-fetching]]
- [[router-internals]]
- [[06-software-design/system-design/caching|Caching]]
- [[06-software-design/api-design/idempotency|Idempotency]]
