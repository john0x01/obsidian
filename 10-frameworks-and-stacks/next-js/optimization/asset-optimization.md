# Asset Optimization

Next.js ships a set of components and build behaviors whose entire purpose is to keep the *bytes the browser must fetch and the layout the browser must compute* under control: `next/image`, `next/font`, `next/script`, plus automatic code splitting and link prefetching. None of these are conveniences — each one encodes a hard-won web-performance lesson (CLS, render-blocking resources, oversized payloads) directly into the framework so you don't re-discover it per project.

## next/image: Bytes and Layout Stability

`next/image` exists to solve three problems at once: images are usually the heaviest bytes on a page, they're the top cause of layout shift, and they're tedious to make responsive by hand.

- **Responsive sizing.** Given `width`/`height` (or `fill`), Next generates a `srcset` across breakpoints. The `sizes` prop tells the browser how wide the image renders at each viewport so it picks the smallest sufficient candidate. Getting `sizes` wrong is the most common cause of over-downloading — the browser defaults to assuming full viewport width and fetches a too-large source.
- **Modern formats.** The image optimizer transcodes to AVIF/WebP on demand based on the request's `Accept` header (content negotiation), falling back to the original format — smaller bytes for capable clients, no manual pipeline.
- **Lazy loading.** Off-screen images are `loading="lazy"` by default; the browser defers them until near the viewport. For your LCP image you opt out with `priority`, which also preloads it — using `priority` on the hero is one of the highest-leverage LCP fixes.
- **CLS avoidance.** Because you declare dimensions (or `fill` with a sized container), the browser reserves space *before* the image loads, so content doesn't jump. This is the mechanism behind near-zero CLS, not a side effect.

The optimizer runs server-side (a Node-capable image step) and caches output. Pitfalls: unconfigured `remotePatterns` blocks external sources; pointing `next/image` at an already-optimized CDN can double-process; and naive `fill` without a positioned, sized parent collapses layout.

## next/font: Self-Hosting and Zero Layout Shift

Web fonts cause two distinct problems: a network request to a third party (privacy + latency) and *layout shift* when the fallback font is swapped for the real one (FOUT/FOIT). `next/font` attacks both at **build time**:

- **Self-hosting.** Google Fonts (and local fonts) are downloaded at build and served from your own origin — zero runtime requests to Google, no extra DNS/connection, better privacy.
- **Zero layout shift.** This is the clever part: Next computes a **size-adjusted fallback font** using the target font's metrics, applying `size-adjust`/`ascent-override`/`descent-override` so the fallback occupies almost exactly the same space. When the real font swaps in, glyphs barely move — CLS approaches zero by construction, not by `font-display` luck.
- The font is loaded as a CSS variable / className you attach, with subsetting to ship only the glyph ranges you use.

The mental model: fonts become a *build-time asset with precomputed metrics*, not a runtime third-party dependency.

## next/script: Controlling Third-Party JS

Third-party scripts (analytics, tag managers, chat widgets) are the silent killers of TBT and hydration timing. `next/script` gives you a `strategy` to place each one in the lifecycle deliberately:

- `beforeInteractive` — loads before hydration; reserve for genuinely critical scripts (rare; e.g. consent/bot-detection that must run first).
- `afterInteractive` (default) — loads after the page becomes interactive; the right home for most analytics.
- `lazyOnload` — loads during idle time after everything else; for low-priority widgets (chat, social).
- `worker` (experimental, Partytown-based) — offloads the script to a web worker, keeping the main thread free.

The philosophy: you almost never want third-party JS competing with hydration, so the framework makes priority an explicit, per-script decision.

## Code Splitting and Link Prefetching

Two behaviors are automatic and worth understanding rather than ignoring:

- **Automatic code splitting.** Each route gets its own JS bundle; shared modules are factored into common chunks. You don't ship the whole app to load one page. `next/dynamic` (and React `lazy`) let you split *within* a route — defer a heavy modal/editor until it's needed, optionally with `ssr: false` for client-only widgets.
- **Link prefetching.** `<Link>` prefetches the linked route's payload when the link enters the viewport (in production). In the App Router this prefetch is RSC-aware: it can fetch the route's React Server Component payload so navigation feels instant. Prefetch behavior is configurable (`prefetch` prop), and its exact defaults — especially around static vs dynamic segments and partial prefetching — have shifted across versions, so verify the precise semantics for your Next.js 15 minor rather than assuming the old "always full prefetch."

```tsx
import dynamic from 'next/dynamic'
// ship the editor only when this component actually renders
const Editor = dynamic(() => import('./editor'), { ssr: false })
```

## Senior Pitfalls & Philosophy

- **`sizes` is mandatory thinking, not optional polish** — wrong `sizes` silently wastes bandwidth on every device.
- **`priority` exactly once**, on the LCP image; sprinkling it everywhere re-creates the render-blocking problem it solves.
- **Don't double-optimize** images already served by an image CDN.
- **Audit every third-party script's strategy** — the default `afterInteractive` is right for analytics but wrong for a heavy widget that should be `lazyOnload`.
- **Prefetch can over-fetch** on link-dense pages; measure and tune `prefetch` rather than trusting defaults blindly.

The unifying idea: Next bakes the Core Web Vitals failure modes (LCP from heavy/late images, CLS from un-sized media and font swaps, TBT from third-party JS) into framework primitives so the *default* path is the performant one — your job is to not opt out of it accidentally.

## See Also

- [[turbopack-and-swc]]
- [[11-applied/web-platform/core-web-vitals|Core Web Vitals]]
- [[11-applied/web-platform/critical-rendering-path|Critical Rendering Path]]
- [[07-performance-engineering/frontend-performance|Frontend Performance]]
- [[11-applied/web-platform/content-negotiation|Content Negotiation]]
