# Senior Next.js Self-Test

A self-assessment for mastery, not recall. If you can answer these out loud with mechanism and trade-offs — not just API names — you understand Next.js 15 at a senior level.

## Rendering Strategies

- What is the precise difference between static rendering, dynamic rendering, and streaming in the App Router, and what triggers Next to switch a route from static to dynamic?
- How does Partial Prerendering (PPR) change the static/dynamic boundary, and what does it let you ship that pure SSG or pure SSR cannot?
- Explain how `<Suspense>` boundaries shape the streamed HTML response and what the user perceives at each flush.
- When does Incremental Static Regeneration (ISR) revalidate, and what is the difference between time-based and on-demand revalidation in terms of who pays the latency?
- Why can reading `cookies()`, `headers()`, or `searchParams` opt a route out of static rendering, and how do you reason about that boundary deliberately?
- What is the difference between SSG, SSR, and ISR in terms of *when* HTML is produced and *where* the cost lands?
- How would you decide, for a given page, between fully static, ISR, and fully dynamic — what signals drive that choice?

## App Router

- What exactly is the difference between a Server Component and a Client Component, and what does the `'use client'` directive mark — the file, or a boundary?
- How does the file-system router map `layout`, `page`, `template`, `loading`, `error`, and `not-found` to the actual render tree and lifecycle?
- Why do layouts persist across navigations while templates remount, and when does that distinction matter?
- How do Server Actions work end-to-end — what does the client actually send, and how does Next route it to the server function?
- What are parallel routes and intercepting routes, and what UX problems do the `@slot` and `(.)`/`(..)` conventions solve?
- How does data flow from a Server Component to a Client Component, and why can't you pass a function or non-serializable value across that boundary?
- What is the difference between Route Handlers (`route.ts`) and Server Actions, and when would you reach for each?

## Caching & Data

- Name the distinct caching layers Next 15 maintains and what each one keys on and invalidates on.
- How did the default caching behavior for `fetch` and Route Handlers change in Next.js 15, and what does "uncached by default" force you to do explicitly?
- What is the Full Route Cache versus the Data Cache versus the Router Cache (client-side), and how do they interact during a navigation?
- How do `revalidateTag` and `revalidatePath` differ in scope and blast radius, and how do they relate to the `next: { tags }` fetch option?
- When does the client-side Router Cache serve stale UI, and how do you force a fresh fetch after a mutation?
- What is request memoization within a single render pass, and why does it let you call the same `fetch` in multiple components without duplicate requests?
- How would you debug a page that "won't update" after you changed the underlying data — which cache do you suspect first and how do you confirm?

## RSC Internals

- What is the React Server Component payload (the RSC wire format), and how does it differ from sending HTML or JSON?
- How does hydration work when the tree is a mix of server-rendered and client components — what does React reconcile and what does it skip?
- What does "serialization boundary" mean for props crossing from server to client, and what are the consequences for bundle size?
- Why are Server Components zero-JS-to-the-client by default, and how does that change how you think about where logic lives?
- How does streaming RSC interleave with `Suspense` to send the shell first and stream slots as data resolves?
- What actually ends up in the client bundle for a given route, and how do you trace why a "server" dependency leaked into it?

## Runtime & Middleware

- What APIs are available in the Edge runtime versus the Node.js runtime, and what is the underlying execution model that explains the difference?
- Why does the Edge runtime have near-zero cold starts, and what do you give up to get them?
- Where does Middleware sit in the request lifecycle, and why does "runs before the cache" both empower and constrain it?
- What does the `matcher` config actually do, and why must its patterns be statically analyzable?
- What are the four things a `NextResponse` can do in middleware, and how do `rewrite` and `redirect` differ from the user's perspective?
- Why is doing full authorization (DB-backed permission checks) in middleware an anti-pattern, and where should that logic live instead?
- How do you choose `runtime = 'edge'` versus `'nodejs'` for a Route Handler, and what concrete factors (data location, deps, CPU, cold start) drive the decision?

## Optimization

- How does `next/image` prevent layout shift, and what is the mechanism behind reserving space before the image loads?
- What does the `sizes` prop control, and how does getting it wrong cause over-downloading even when `srcset` is correct?
- How does `next/font` achieve near-zero CLS for web fonts — what is the size-adjusted fallback and what CSS descriptors implement it?
- What do the `next/script` strategies (`beforeInteractive`, `afterInteractive`, `lazyOnload`, `worker`) each mean for the page lifecycle, and which fits analytics versus a chat widget?
- How does automatic code splitting differ from `next/dynamic`, and when do you reach for `ssr: false`?
- How does `<Link>` prefetching work in the App Router, and why is RSC-aware prefetching what makes navigation feel instant?
- Why is SWC disabled the moment a `babel.config.js` exists, and how would you detect that you've accidentally fallen off the SWC fast path?
- What does Turbopack's function-level incremental cache buy you, and why does rebuild cost track change size rather than codebase size?
