# Route Handlers

A Route Handler is a `route.ts` (or `route.js`) file inside `app/` that defines a **custom HTTP endpoint** using the Web platform's `Request`/`Response` model. It is the App Router's replacement for `pages/api/*`, but built on Web standards (`Request`, `Response`, `Headers`, `URL`) rather than the Node `(req, res)` signature. The core mental model: you export named functions, one per HTTP verb, each receiving a `Request` and returning a `Response`.

## The Shape

You export functions named after HTTP methods — `GET`, `POST`, `PUT`, `PATCH`, `DELETE`, `HEAD`, `OPTIONS`:

```ts
// app/api/posts/route.ts
export async function GET(request: Request) {
  const { searchParams } = new URL(request.url)
  const q = searchParams.get('q')
  const data = await db.posts.search(q)
  return Response.json(data)          // Web Response, not res.json()
}

export async function POST(request: Request) {
  const body = await request.json()
  const created = await db.posts.create(body)
  return Response.json(created, { status: 201 })
}
```

There is **no `res` object.** You construct and return a `Response` (or `NextResponse`, a thin superset that adds cookie/redirect helpers). For dynamic segments the handler receives a second argument with `params` (a **Promise** in Next 15): `async function GET(req, { params }) { const { id } = await params }`. A segment with a `route.ts` **cannot also have a `page.tsx`** — UI and endpoint are mutually exclusive at the same path.

## The Web Request/Response Model

This is the deeper point: Route Handlers are deliberately framework-agnostic. `request` is a standard `Request`, so `request.json()`, `request.formData()`, `request.headers.get()`, and `request.url` are the same APIs you'd use in a Service Worker, Cloudflare Worker, or Deno. This is what lets handlers run on the **Edge runtime** (`export const runtime = 'edge'`) as well as Node. For cookies and headers Next also exposes the async `cookies()` and `headers()` helpers from `next/headers`, which is the idiomatic way to read request context.

## Caching Behavior

This is the part that trips up engineers migrating from `pages/api` (which was never cached). In the App Router:

- **`GET` handlers are dynamic by default in Next 15.** Earlier versions cached `GET` aggressively; Next 15 flipped the default so handlers are *not* cached unless you opt in. Any handler reading `request`, `cookies()`, `headers()`, or using dynamic functions is dynamic regardless.
- **Opt into caching** with route segment config: `export const dynamic = 'force-static'`, or `export const revalidate = 3600` to cache and revalidate on a time window.
- **Non-`GET` methods** (`POST`, etc.) are **always dynamic** — never cached.

So the rule of thumb: a handler is dynamic unless you explicitly mark it static *and* it touches no request-specific data.

## When to Use Handlers vs Server Actions

This is the genuinely important architectural decision in Next 15. Both let you run server code in response to client interaction, but they serve different purposes:

- **Use Route Handlers when you need a real, addressable HTTP endpoint:**
  - Webhooks (Stripe, GitHub) that external systems POST to.
  - Public/third-party APIs, OAuth callbacks, anything consumed outside your own React tree.
  - Serving non-JSON responses: files, images, RSS/`sitemap.xml`, streaming responses, `Content-Type`-specific output.
  - Mobile or other non-browser clients hitting your backend.
- **Use Server Actions (`'use server'`) for mutations originating from *your own* UI:**
  - Form submissions and button-triggered mutations from your components.
  - Progressive enhancement — actions work without JS, bind to `<form action={...}>`.
  - Integrated with `revalidatePath`/`revalidateTag` and `useFormStatus`/`useActionState` so the page updates after the mutation with no manual fetch wiring.

The heuristic: **if the caller is your own React tree, prefer a Server Action; if the caller is anything else (external system, generic HTTP client) or you need a specific response shape, use a Route Handler.** Server Actions remove the boilerplate of writing an endpoint, serializing, fetching it, and re-fetching data — but they only make sense when both ends are yours.

## Senior Pitfalls

- **Assuming `GET` handlers are cached like in older Next** — in 15 they're dynamic by default; don't rely on stale caching behavior, and don't assume your data is cached.
- **Reaching for a Route Handler to mutate your own app's data** when a Server Action would eliminate the fetch/serialize/revalidate round-trip.
- **Using Route Handlers as a `getServerSideProps` substitute** — Server Components already fetch on the server; you rarely need an internal API the page then fetches from itself (a self-`fetch` waterfall).
- **Forgetting `params` is a Promise** in Next 15 dynamic handlers.
- **Putting secrets in a handler exported as `GET` and forgetting it's a public URL** — handlers are world-reachable; auth is your responsibility (check `cookies()`/headers, no implicit protection).

## See Also

- [[file-conventions]]
- [[server-and-client-components]]
- [[app-router-model]]
- [[advanced-routing]]
