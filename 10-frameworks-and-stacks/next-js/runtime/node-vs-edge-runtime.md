# Node.js Vs Edge Runtime

Next.js can execute your server code under two distinct runtimes: the full **Node.js runtime** and a constrained **Edge runtime** built on Web-standard APIs. The choice is not cosmetic — it determines which APIs exist, how a function cold-starts, where it physically runs, and what failure modes you inherit. Understanding it well is the difference between "my middleware crashes in production but not locally" and shipping with intent.

## Two Runtimes, One Mental Model

The Edge runtime is **not Node.js with fewer features** — it is a different execution environment modeled on the WinterCG / Web-interoperable runtime spec, the same family that powers Cloudflare Workers and Deno Deploy. On Vercel it runs on V8 isolates rather than full Node processes. An isolate is a lightweight V8 sandbox: many isolates share one OS process, so there is no per-invocation process spin-up. That is the entire reason Edge cold starts are near-zero (single-digit ms) while a Node serverless function must boot a Node process, resolve `require`/`import` graphs, and JIT-warm — tens to hundreds of ms cold.

```
Node serverless fn        Edge function (isolate)
─────────────────         ───────────────────────
boot Node process    →    isolate already resident
load module graph    →    snapshot/eval once, reused
JIT warmup           →    shared V8, minimal warmup
heavy cold start          ~0ms cold start
```

The trade is capability. Isolates expose **Web APIs** — `fetch`, `Request`, `Response`, `Headers`, `URL`, `URLPattern`, `TextEncoder`, `crypto` (Web Crypto / `crypto.subtle`), `ReadableStream`, `structuredClone` — but **no Node built-ins**: no `fs`, no `net`/`dgram`, no `child_process`, no `Buffer`-centric native modules, no raw TCP. You cannot open a database socket directly; you talk to data over HTTP (e.g. an HTTP-based DB driver, an edge-friendly client). Many npm packages that `require('fs')` or ship native `.node` addons simply will not bundle for Edge. Node APIs that *are* polyfilled exist as a curated subset, not the whole stdlib.

## Where Each Runs and Its Limits

- **Node runtime**: regional serverless functions (or long-lived servers when self-hosted). Generous memory, larger bundle/uncompressed-size budgets, longer max durations, full filesystem and process access. This is the default for Route Handlers, Server Components rendering, and Server Actions.
- **Edge runtime**: globally distributed, runs close to the user. Tight CPU-time budgets per request and small bundle-size ceilings (kilobytes-to-low-MB range depending on host). On Vercel the function size and per-request CPU are deliberately constrained because isolates are multi-tenant and must stay light.

Concrete limits are **host-specific and version-specific** — Vercel, Cloudflare, and self-hosted Node differ, and the numbers shift. Treat the exact MB/second figures as "look it up for your deploy target," not memorized constants. The *shape* is stable: Edge = small + fast-cold + globally near + capability-limited; Node = large + slow-cold + regional + full-capability.

## Streaming and the Web-Standard Model

Both runtimes stream responses, but the Edge model is streaming-native: a Route Handler can return a `Response` wrapping a `ReadableStream`, and the isolate flushes chunks as they're produced. This pairs naturally with React Server Components streaming and `Suspense`. Because the Edge runtime *is* the Web platform, the same `Response`/`ReadableStream` code you'd write for a Worker runs unchanged. The Node runtime also streams (and underpins RSC streaming for the default rendering path), but you're closer to Node primitives.

## Selecting a Runtime

Per route or middleware you opt in with the segment config:

```ts
// app/api/geo/route.ts  — opt this route into Edge
export const runtime = 'edge'      // or 'nodejs' (the default)
```

Middleware historically runs on Edge by default (see [[middleware]]); Next.js 15 has been generalizing middleware/runtime config, so confirm the flag names for your exact minor version rather than assuming.

**Choose Edge when**: the work is light and latency-to-user dominates — geolocation/header inspection, A/B bucketing, auth-token gate-checks, redirects/rewrites, personalization, simple JSON from HTTP data sources, response streaming from a global PoP.

**Choose Node when**: you need real Node APIs (filesystem, native crypto beyond Web Crypto, native addons), a TCP database driver, heavy CPU (image processing, large parsing), big dependencies, or longer execution. Most data-heavy Server Components and Server Actions belong here.

## Senior Pitfalls

- **"Works locally, breaks deployed."** `next dev` is forgiving; a Node-only import only explodes when Edge bundling rejects it. Test the real runtime.
- **Transitive Node deps.** Your code may be Edge-clean while a third dependency reaches for `fs`. The bundler error points at the leaf, not your intent.
- **Cold-start ≠ free at scale.** Edge avoids process cold starts but still pays for isolate eval of your bundle — bloated bundles erode the advantage and may breach size limits.
- **No connection pooling on Edge.** Stateless HTTP per request; you can't lean on a warm pooled socket. Use HTTP/serverless-oriented drivers and accept the model.
- **Don't reach for Edge reflexively.** "Edge = faster" is folklore. If your data lives in one region, an Edge function that fans out to a distant origin can be *slower* than a co-located Node function. Latency is end-to-end, not first-byte.

The philosophy: Edge trades capability for ubiquity and instant cold start by adopting the Web platform as its API contract; Node trades reach and cold-start cost for the full power of the server. Pick by where the latency and the data actually live.

## See Also

- [[middleware]]
- [[turbopack-and-swc]]
- [[07-performance-engineering/network-performance|Network Performance]]
- [[11-applied/web-platform/cdn|CDN]]
- [[08-quality-and-operations/security/authentication|Authentication]]
