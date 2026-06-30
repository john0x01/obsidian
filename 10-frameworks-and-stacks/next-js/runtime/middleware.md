# Middleware

Next.js Middleware is code that runs **before a request is completed** — after the request hits the platform but before it reaches a route, cache, or rendering. It is the single global interception point in the request lifecycle, and because it sits in front of *everything*, it is both extremely powerful and easy to misuse. By default it executes on the Edge runtime, which dictates what it can and cannot do.

## Position in the Request Lifecycle

```
incoming request
      │
      ▼
┌──────────────┐   matcher decides: run or skip
│  Middleware  │── skip ──▶ normal routing
└──────┬───────┘
       │ NextResponse.next() / rewrite / redirect / respond
       ▼
  routing → cache lookup → render (RSC) / route handler → response
```

Middleware runs on **every matched request before the cache and before rendering**. That placement is the whole point: you can rewrite where a request lands, redirect it elsewhere, or short-circuit with a response — all before Next.js commits to a route. But it also means middleware is on the hot path of *every* matched request, so its cost is paid universally. Keep it cheap.

## What It's For

- **Auth gating**: read a session cookie / token, and redirect unauthenticated users to `/login`. Middleware is ideal for the coarse "is there a plausible session" gate — *not* for full authorization decisions that need a DB round-trip.
- **Redirects**: legacy-URL → new-URL, locale or country redirects, enforcing canonical hosts/trailing slashes.
- **Rewrites**: serve `/dashboard` content from an internal path without changing the visible URL — the basis of proxying, multi-tenant host routing, and incremental migrations.
- **A/B testing & personalization**: bucket a user (cookie/geo/header) and rewrite to a variant. Because it runs before cache, it can vary the served path per cohort.
- **i18n**: detect `Accept-Language` or a locale cookie and rewrite/redirect to the localized segment.

## NextResponse: The Control Surface

Middleware returns a `NextResponse` (a thin extension of the Web `Response`). The four moves:

```ts
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(req: NextRequest) {
  // 1. continue to the intended route
  // return NextResponse.next()

  // 2. rewrite — different content, same URL in the browser
  // return NextResponse.rewrite(new URL('/variant-b', req.url))

  // 3. redirect — browser navigates, URL changes
  // if (!req.cookies.get('session'))
  //   return NextResponse.redirect(new URL('/login', req.url))

  // 4. respond directly — short-circuit
  // return new NextResponse('blocked', { status: 403 })

  // mutate request/response headers passed downstream
  const res = NextResponse.next()
  res.headers.set('x-variant', 'b')
  return res
}
```

`NextRequest` adds ergonomics over the raw `Request`: `req.cookies`, `req.nextUrl` (parsed, mutable URL), and `req.geo`/`req.ip` on platforms that provide them. You can set request headers that flow downstream to the route, and response headers/cookies that flow back to the client.

## The Matcher

By default middleware matches all paths. You scope it with `config.matcher` so it doesn't run on assets and irrelevant routes — this is a performance lever, not a nicety:

```ts
export const config = {
  matcher: [
    // run on everything except static files, image optimizer, and favicon
    '/((?!_next/static|_next/image|favicon.ico).*)',
  ],
}
```

The matcher is evaluated *before* the function runs — non-matching requests pay zero middleware cost. Matchers can be path arrays or objects with `source` and conditions (e.g. header/cookie `has`/`missing`). They are statically analyzed, so they must be literal patterns, not runtime-computed strings.

## Limitations

- **Edge runtime by default**: no `fs`, no native Node modules, no raw DB sockets, tight CPU budget, small bundle (see [[node-vs-edge-runtime]]). Talk to data over HTTP or defer it to the route. Next.js has been evolving runtime options, so verify whether your version supports a Node middleware runtime before relying on Node APIs.
- **One middleware file**: a single `middleware.ts` at the project root (or `src/`). You compose logic *inside* it; you don't get a chain of files.
- **Runs before cache** — heavy work here defeats caching for the whole app. Resist turning middleware into a mini-API.
- **No rendering**: it routes and responds; it doesn't render React.

## Senior Pitfalls & Philosophy

- **Don't do real authorization in middleware.** A cookie's *presence* is a cheap gate; verifying a JWT signature is borderline-OK on Edge via Web Crypto, but DB-backed permission checks belong in the route/layer that owns the data. Middleware that calls your DB on every request is an anti-pattern.
- **Redirect loops** are the classic self-inflicted outage — a matcher that catches `/login` and redirects unauthenticated users to `/login` loops forever. Exclude your auth pages from the gate.
- **Cookie writes for sessions**: when refreshing tokens, set cookies on the *response* you return *and* propagate to the request so the downstream render sees them — a frequent subtle bug.
- **Latency tax**: every matched request pays middleware's runtime cost before anything else. Keep it sub-millisecond in spirit; profile it like the hot path it is.

The mental model: middleware is your programmable reverse-proxy edge layer — fast, global, capability-limited, and on the critical path of everything it matches. Use it to *route and gate*, not to *compute*.

## See Also

- [[node-vs-edge-runtime]]
- [[08-quality-and-operations/security/authentication|Authentication]]
- [[08-quality-and-operations/security/sessions-and-cookies|Sessions and Cookies]]
- [[08-quality-and-operations/devops-and-infrastructure/feature-flags|Feature Flags]]
- [[11-applied/web-platform/cdn|CDN]]
