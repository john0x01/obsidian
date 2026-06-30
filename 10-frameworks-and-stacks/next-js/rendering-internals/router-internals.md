# App Router Client Runtime

The App Router ships a client-side runtime whose job is to make navigation feel like a single-page app while the actual UI is described by server-rendered RSC payloads. It does this through **soft navigation**, **prefetching**, and an in-memory **Router Cache** of RSC payloads. The mental model: the client holds a tree of route segments, and navigating means fetching the RSC payload for the segments that changed and reconciling them into the live tree — never reloading the document.

## Segments and the segment tree

A route is a path of nested **segments** (each `layout`/`page`/`template` level). The router models the current route as a tree of these segments and tracks, per segment, its cached RSC payload. Layouts are *shared* across navigations that don't change them — moving from `/dashboard/a` to `/dashboard/b` keeps the `dashboard` layout (and any Client Component state inside it) mounted, swapping only the leaf. This selective sub-tree replacement is the core of why App Router navigation preserves scroll, focus, and client state.

## Soft navigation

A **soft navigation** is an in-app transition (via `<Link>` or `router.push`) that does **not** reload the page. On click, the runtime:

1. Determines which segments differ between current and target routes.
2. Fetches the RSC payload for the changed segments (or reads it from the Router Cache / prefetch buffer).
3. Reconciles the new payload into the existing React tree, re-rendering only the changed subtree.

Because it's a tree reconciliation of serialized React elements (see [[rsc-and-flight]]), Client Components outside the changed segments are untouched and keep their state. A **hard navigation** (full reload, typing a URL, `location.assign`) discards everything and re-fetches HTML + payload from scratch.

## Prefetching

`<Link>` prefetches by default when it enters the viewport (and on hover/touch for some cases). Prefetching fetches the target's RSC payload ahead of the click and stores it in the Router Cache so the navigation is instant. What gets prefetched depends on staticness:

- **Static routes** prefetch the full payload.
- **Dynamic routes** historically prefetched only down to the nearest `loading.js` boundary (the shared layout shell), so the click shows an instant skeleton and the dynamic part streams. You can force full prefetch with `prefetch={true}` or disable with `prefetch={false}`.

Prefetch is a latency optimization, not a correctness mechanism — prefetched payloads are subject to the Router Cache's staleness rules below.

## The Router Cache

The Router Cache (a.k.a. client-side cache) is an **in-memory, per-session** store of RSC payloads keyed by segment. It is the layer most responsible for "why is my page stale after I navigate back to it." Its key properties in **Next 15**:

- It caches visited and prefetched segment payloads for the session; it's wiped on full page reload.
- The default **`staleTime` for page segments is `0`** in Next 15 — pages are re-fetched on navigation rather than served stale from this cache (a deliberate reversal of the Next 14 default, which cached pages for ~30s and surprised many people with stale screens). Prefetched/static and layout data still get a short reuse window.
- It is invalidated by: `router.refresh()`, a Server Action response, and server-side `revalidatePath`/`revalidateTag` (which propagate a signal so the client drops the relevant entries). `router.refresh()` re-fetches the current route's payload from the server while preserving client state.

```
click <Link href="/b">
   │  segments differ at leaf only
   ▼
Router Cache hit? ──yes──▶ reconcile cached payload into tree ─▶ done (instant)
   │ no / stale
   ▼
fetch RSC payload for changed segments  (POST/GET to server, RSC stream)
   ▼
reconcile into existing tree; layouts above the change stay mounted
```

## History and back/forward

Soft navigations use the History API (`pushState`/`replaceState`); back/forward are intercepted via `popstate`. Back-forward navigation reads from the Router Cache when possible so it's instant and **restores scroll position**. This is why BFCache-like behavior works without a full reload: the payloads (and the React tree state for preserved segments) are reconstructed from cache. The trade-off is the staleness question — if data changed since you last viewed a route, back-navigation might show the cached payload; the Next 15 `staleTime: 0` default for pages makes this far less common than in 14.

## Senior pitfalls and misconceptions

- **"Navigation fetches HTML."** Only the *first* load and hard navigations do. Soft navigations fetch RSC payloads and reconcile — that's the whole point.
- **Stale UI after a mutation** is almost always the Router Cache holding an old payload on the client, even though the server's Data Cache is fresh (see [[caching-layers]]). Ensure the mutation path triggers `revalidatePath`/`revalidateTag` or `router.refresh()` so the client invalidates.
- **Carrying Next 14 staleness intuitions into 15.** The 30s page cache that caused "I clicked away and back and saw old data" is gone by default; reasoning from the old behavior will mislead you.
- **Over-relying on prefetch for freshness.** Prefetched payloads can be stale; prefetch optimizes latency, not correctness. Pair with proper revalidation.
- **`router.refresh()` is not a reload.** It re-fetches server data and re-renders Server Components while preserving Client Component state and the DOM — strictly better than `location.reload()` when you just need fresh server data.

## See Also

- [[caching-layers]]
- [[rsc-and-flight]]
- [[revalidation]]
- [[client-server-boundary]]
- [[server-actions]]
- [[11-applied/web-platform/critical-rendering-path|Critical Rendering Path]]
