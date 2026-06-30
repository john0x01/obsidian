# Partial Prerendering

Partial Prerendering (PPR) dissolves the false choice between "the whole route is static" and "the whole route is dynamic." A PPR route ships a **prerendered static shell** instantly from the cache/CDN, and the genuinely dynamic pieces — the "holes" — are **streamed into that shell within the same HTTP response**. One route, one request, both rendering modes. It is the architectural endgame of the App Router's per-segment philosophy, and as of Next.js 15 it remains **experimental**.

## The Problem It Targets

Before PPR you faced a coarse trade-off: a single dynamic dependency (a personalized cart count, an auth-gated banner) forced the *entire* route dynamic, sacrificing the cache-served TTFB of the 90% of the page that was static. The usual workaround — wrap the dynamic bit in Suspense and stream it — still left the route classified dynamic and computed at request time. PPR makes the static majority truly *prerendered* while keeping the dynamic minority dynamic.

## The Mental Model: Holes In A Static Shell

You author the route normally. The dynamic parts are whatever reads request-time input (`cookies()`, `headers()`, `searchParams`) — and you wrap them in `<Suspense>`:

```tsx
export default function Page() {
  return (
    <main>
      <StaticHero />                 {/* prerendered into the shell */}
      <ProductGrid />                {/* static, prerendered */}
      <Suspense fallback={<CartSkeleton/>}>
        <Cart />                     {/* reads cookies() → a dynamic hole */}
      </Suspense>
    </main>
  )
}
```

At build time Next prerenders everything it can. When it reaches the Suspended subtree that touches dynamic APIs, it records a **hole**: it emits the *fallback* into the static shell and marks the boundary as needing runtime resumption. The shell — hero, grid, fallback skeleton — is a fully static artifact in the Full Route Cache.

## The Prerender → Resume Model

PPR's defining idea is that a render can be **paused at build and resumed at request**:

```
BUILD TIME
  render the tree → hit a dynamic Suspense boundary → emit fallback,
  serialize a "postponed" state describing the unfinished holes → store shell + postponed state

REQUEST TIME
  serve the static shell instantly (TTFB ≈ static)
  RESUME the saved render from the postponed state, computing only the holes
  stream the holes' real HTML into the open response, swapping the fallbacks
```

This is why PPR is "free" for the static part: the shell isn't recomputed per request, only the holes are. React's prerender APIs (the `prerender`/`resume` family that underpin this) capture the postponed state so the runtime can pick up exactly where build left off, rather than re-rendering from scratch. The single response therefore contains a static prefix followed by streamed dynamic chunks — the union of SSG and streaming SSR.

## Combining Static And Dynamic In One Response

The wire result reads like streaming SSR, but the *first chunk is a cached static document*, not a freshly computed one:

```
[from cache, instant] full shell with <CartSkeleton/> placeholder
[streamed at request]  real <Cart/> HTML + swap script for the hole
```

So you get static-grade TTFB and FCP for the shell, plus correct per-request content for the holes, with no route-wide dynamic penalty. The dynamic holes still pay their normal cost — but only the holes.

## Relation To dynamicIO And Current Status

PPR is entangled with a broader caching rework. In Next 15 the `experimental.ppr` flag enables it (often paired with `export const experimental_ppr = true` per route to adopt incrementally). It sits alongside the experimental **`dynamicIO`** model, which inverts caching defaults — making I/O dynamic unless you explicitly mark it cacheable (e.g. with a `use cache` directive and `cacheLife`/`cacheTag`). The two are designed to converge: `dynamicIO` gives a precise, opt-in definition of what is static vs. dynamic, which is exactly the boundary information PPR needs to know what to bake into the shell versus leave as a hole.

These APIs are **moving targets**. The flag names, the `use cache` directive, and whether PPR ships stable or folds into a renamed "cache components" model have been in flux across 15.x and into the Next 16 line. Treat specific flag/directive names as version-pinned: verify against the docs for your exact Next version rather than memorizing them. What is *stable* and worth internalizing is the **prerender→resume concept** — that survives whatever the surface API ends up being called.

## Senior Pitfalls And Philosophy

- **It's experimental — don't ship it load-bearing without an exit plan.** Behavior, flags, and the surrounding cache model are unstable in 15.x.
- **Holes are defined by *where you put Suspense*, not by intent.** If a dynamic API read isn't enclosed in a Suspense boundary, the build can't isolate it as a hole — it errors or demotes the route. The boundary *is* the static/dynamic contract.
- **A dynamic read that escapes Suspense breaks the shell.** Reading `cookies()` directly in the page body (outside any boundary) makes the whole thing dynamic again — PPR only helps if the dynamism is *contained*.
- **Don't confuse PPR with ISR.** ISR refreshes a *fully static* page over time; PPR mixes static and *per-request* dynamic content in one render. They solve different axes (freshness vs. composition) and can stack.

Philosophy: the response should be the *finest-grained* blend of cached and computed content the page actually requires — static where it can be, dynamic only at the precise holes that need it, delivered as one stream. PPR is the App Router's per-segment thesis pushed below the route, down to the individual Suspense boundary.

## See Also
- [[rendering-strategies]]
- [[static-and-isr]]
- [[streaming-ssr]]
- [[hydration]]
- [[10-frameworks-and-stacks/react/fiber/fiber-architecture|React Fiber Architecture]]
