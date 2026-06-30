# Next.js

Deep notes on Next.js — rendering strategies, the App Router, the caching layers, and the RSC/streaming internals. Targets Next.js 15 (App Router first). For mastery, not API lookup.

[← Back to Frameworks & Stacks](../README.md)

## Rendering
- [Rendering Strategies](rendering/rendering-strategies.md) — CSR/SSR/SSG/ISR/streaming/PPR and when each runs
- [Static & ISR](rendering/static-and-isr.md)
- [Streaming SSR](rendering/streaming-ssr.md)
- [Partial Prerendering](rendering/partial-prerendering.md) — static shell + dynamic holes
- [Hydration](rendering/hydration.md)

## App Router
- [App Router Model](app-router/app-router-model.md) — server-first, nested layouts, the route tree
- [File Conventions](app-router/file-conventions.md) — layout/page/loading/error/route
- [Server & Client Components](app-router/server-and-client-components.md) — the `'use client'` boundary
- [Route Handlers](app-router/route-handlers.md)
- [Advanced Routing](app-router/advanced-routing.md) — parallel, intercepting, route groups

## Data & Caching
- [Data Fetching](data-and-caching/data-fetching.md) — fetching in Server Components, waterfalls
- [Caching Layers](data-and-caching/caching-layers.md) — request memoization, Data Cache, Full Route Cache, Router Cache
- [Revalidation](data-and-caching/revalidation.md) — time/tag/path, the Next 15 defaults
- [Server Actions](data-and-caching/server-actions.md)

## Rendering Internals
- [RSC & the Flight Protocol](rendering-internals/rsc-and-flight.md)
- [Client/Server Boundary](rendering-internals/client-server-boundary.md)
- [Router Internals](rendering-internals/router-internals.md) — soft navigation, prefetch, Router Cache

## Runtime & Build
- [Node vs Edge Runtime](runtime/node-vs-edge-runtime.md)
- [Middleware](runtime/middleware.md)
- [Turbopack & SWC](compilation/turbopack-and-swc.md)
- [Asset Optimization](optimization/asset-optimization.md) — `next/image`, `next/font`, prefetching

## Pages Router (legacy)
- [Pages Router](pages-router/pages-router.md) — `getServerSideProps`/`getStaticProps`, migration

## Self-Test
- [Review Questions](review-questions.md)
