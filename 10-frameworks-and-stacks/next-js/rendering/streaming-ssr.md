# Streaming SSR

Streaming SSR is about *delivery*, not freshness: instead of computing the whole page on the server and sending one HTML blob when the slowest data resolves, the server sends HTML in **chunks** as parts of the tree become ready. The browser paints the fast parts immediately and fills in the slow parts as their chunks arrive. In the App Router this is the default behavior wired to React 18+'s `renderToReadableStream`, Suspense boundaries, and (with RSC) a second stream of serialized component data.

## The Old Failure Mode It Fixes

Classic SSR is "all-or-nothing": TTFB is gated by your *slowest* server-side data dependency, because the response can't begin until `renderToString` finishes the entire tree. One slow database call holds the entire document hostage — the user stares at a white screen. Streaming breaks the response into independently flushable units so a slow region no longer blocks the fast ones.

## The Mechanism: Suspense As The Chunk Boundary

A `<Suspense>` boundary is the unit of streaming. React renders everything it can synchronously, and wherever a child *suspends* (throws a promise — what `await` in a Server Component or a suspending data read does), React:

1. Immediately emits the boundary's `fallback` into the initial HTML stream, wrapped in a placeholder with an ID.
2. Continues streaming the rest of the document around it.
3. When the suspended subtree resolves, emits its real HTML in a later chunk **plus a tiny inline `<script>`** that swaps the placeholder for the real content (using `<template>` + a runtime that React inlines).

```
[initial flush] <header/> <Suspense fallback=skeleton/> <footer/>   ← painted fast
       ...time passes, DB resolves...
[later chunk]   <template id="B:0">...real list...</template><script>$RC(...)</script>
```

The order on the wire is "as resolved," but the DOM ends up correct because each chunk carries instructions to slot itself into place. This is **out-of-order streaming**, and it's why a slow widget at the top doesn't block a fast widget at the bottom.

## loading.tsx Is Sugar For A Suspense Boundary

`app/.../loading.tsx` is not a special primitive — Next wraps the segment's `page` in `<Suspense fallback={<Loading/>}>` for you. That's the whole feature. It gives instant route-transition feedback because the shell (layout) streams immediately while the page content streams behind the boundary. You can place explicit `<Suspense>` boundaries *inside* a page for finer-grained streaming than `loading.tsx` (which is per-segment).

```tsx
// loading.tsx ≈ wrapping page in Suspense at the segment level
<Suspense fallback={<Skeleton/>}>
  <SlowServerComponent />
</Suspense>
```

## How It Composes With RSC

With React Server Components there are actually *two* things streaming over one HTTP response, interleaved:

- The **HTML stream** — for first paint and no-JS.
- The **RSC payload** (the serialized "Flight" stream) — a compact description of the rendered Server Component tree, including references to Client Components and `Promise`s for not-yet-resolved Suspense content.

Server Components run to completion on the server and never ship as JS; only their *output* streams. Suspended segments stream as later rows in the Flight payload, which the client reconciles. So streaming and RSC are deeply intertwined: RSC defines *what* serializes, streaming defines *when* it arrives.

## Selective Hydration

Streaming's payoff on the client is **selective hydration**: React 18 can hydrate boundaries independently and out of order as their HTML arrives, and it *prioritizes* hydration of whatever the user interacts with first. If a user clicks a not-yet-hydrated island, React hydrates that boundary ahead of the queue (synchronously enough to handle the event). This decouples interactivity from "the whole page finished loading" — the precondition for it is exactly the Suspense boundaries you placed for streaming.

## The Real Benefits

- **TTFB drops** — the response starts as soon as the shell is ready, independent of slow data.
- **Progressive paint / better FCP & LCP** — meaningful content appears in stages instead of one late flash.
- **Parallel data fetching** — sibling Suspense boundaries let their server data resolve concurrently rather than serially blocking one render.
- **No client waterfall for above-the-fold** — fast content isn't held back by slow content.

## Senior Pitfalls And Philosophy

- **Streaming trades a worse total-load time for a far better *perceived* time.** Total bytes/time may rise slightly (chunk overhead, the inline swap scripts); the win is psychological — users see progress. Don't measure it purely by "fully loaded."
- **`await`-ing data *before* a Suspense boundary blocks the stream.** If you `await` at the top of a Server Component with no boundary below the slow work, you've recreated all-or-nothing SSR. Put the slow await *inside* a Suspended child.
- **Boundaries are also error boundaries' partners.** Pair `<Suspense>` with `error.tsx`/error boundaries; a throw in a streamed segment after headers are sent can't change the HTTP status — you must handle it in-stream.
- **Layout shift from fallbacks.** Skeletons that don't match final dimensions cause CLS when the real content swaps in. Size fallbacks deliberately.
- **Don't over-fragment.** A Suspense boundary per tiny element produces janky staged paints and script chatter. Draw boundaries around *meaningful, independently-slow* regions.

Philosophy: stop thinking of a server response as a single document produced at one instant. It's a *timeline* of progressively-revealed regions, and Suspense is your authoring tool for deciding what the user sees first and what is allowed to arrive late.

## See Also
- [[rendering-strategies]]
- [[static-and-isr]]
- [[partial-prerendering]]
- [[hydration]]
- [[10-frameworks-and-stacks/react/rendering/suspense|React Suspense]] — the boundary primitive
