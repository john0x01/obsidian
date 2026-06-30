# The Pages Router

The Pages Router is the original Next.js routing and data-fetching system, still fully supported in Next 15 and coexistable with the App Router in the same app. Understanding it deeply matters for two reasons: most production Next codebases still contain it, and its design constraints are precisely *what the App Router was built to overcome*. The core model is simple and flat: **every file in `pages/` is a route, every route is a single React component, and data is fetched by exporting one of three special functions alongside it.**

## The Routing Model

`pages/index.tsx` → `/`; `pages/blog/[slug].tsx` → `/blog/:slug` (params via `useRouter().query` or the data-fetching function's `context.params`); `pages/blog/[...slug].tsx` → catch-all. The default export *is* the page. There is no nested-layout primitive — the entire route is one component tree rendered fresh on each navigation, with two global wrappers.

## The Two Special Wrappers

- **`pages/_app.tsx`** — wraps **every** page; receives `{ Component, pageProps }`. The one place for global providers, global CSS, and app-wide layout. It persists across client navigation, so it's where shared state lives.
- **`pages/_document.tsx`** — customizes the server-rendered HTML *document* shell (`<html>`, `<head>`, `<body>`). Renders **once on the server only**, never on the client; you cannot use hooks or handlers here. Used for lang attributes, font preloads, third-party `<script>` placement.

Nested/persistent layouts were never first-class: the community pattern was a per-page `Page.getLayout` function read in `_app`. The App Router's `layout.tsx` is the structural fix for exactly this gap.

## The Data-Fetching Model

This is the heart of the Pages Router, and where its philosophy diverges most from the App Router. Data is fetched by **page-level, opt-in functions** — all-or-nothing per route, run *outside* the component:

- **`getStaticProps`** (SSG) — runs at **build time** (and on revalidation), returns `{ props }` to the page. Add `revalidate: n` for **ISR** (Incremental Static Regeneration): the page is regenerated in the background at most every `n` seconds. Pair with **`getStaticPaths`** for dynamic routes to enumerate which paths to pre-build, with `fallback: false | true | 'blocking'` controlling on-demand generation of un-listed paths.
- **`getServerSideProps`** (SSR) — runs at **request time** on every request, returns `{ props }`. Has access to the raw Node `context` (`req`, `res`, `query`, `params`). Use for per-request, non-cacheable, personalized data. The page cannot be statically optimized.
- **`getInitialProps`** (legacy) — the original API, runs on **both server and client** depending on navigation. Discouraged because it disables automatic static optimization app-wide and blurs the server/client boundary. Prefer the two above.

Two structural properties define this model and its limits:

1. **Page-level granularity.** Data fetching attaches to the whole route, not to individual components. A component deep in the tree that needs data must have it threaded down as props from the page's data function — there is no component-level fetching. This is the prop-drilling and "fat data function" smell of large Pages apps.
2. **All JavaScript ships.** Every component is a client component in the React sense; the page hydrates fully in the browser. There is no server-only component code — your markdown parser, your date library, your data-layer client all contribute to the bundle. This is the bundle-bloat problem RSC eliminates.

## API Routes

`pages/api/*.ts` files export a default handler `(req: NextApiRequest, res: NextApiResponse) => void` — the classic Node-style signature with `res.status().json()`. These are **never cached**, always run server-side, and are the Pages-era backend. The App Router replaces them with Web-standard [[route-handlers]] (`route.ts`, `Request`/`Response`).

## How It Maps Onto / Migrates To the App Router

The two routers can run **side by side** (`pages/` and `app/`) in one project — the recommended incremental migration path. The conceptual mapping:

| Pages Router | App Router |
|---|---|
| `getStaticProps` (+`revalidate`) | `fetch()` with default caching / `revalidate` in a Server Component |
| `getStaticPaths` | `generateStaticParams()` |
| `getServerSideProps` | `async` Server Component reading dynamic APIs (`cookies()`, `headers()`), or `dynamic = 'force-dynamic'` |
| `getInitialProps` | no equivalent — fetch in components |
| `_app.tsx` / `_document.tsx` | root `app/layout.tsx` (owns `<html>`/`<body>`) |
| per-page `getLayout` | nested `layout.tsx` files |
| `pages/api/*` | `app/.../route.ts` Route Handlers |
| `next/head` `<Head>` | the Metadata API (`metadata` / `generateMetadata`) |

The deep shift: data fetching moves from **page-level functions that return props** to **component-level `await` inside Server Components**, and caching moves from a route-level mode to a per-`fetch` / per-segment concern. See [[app-router-model]] for the destination model.

## Senior Pitfalls

- **`getServerSideProps` on routes that didn't need per-request data** — silently opts out of static optimization, hurting performance; many SSR pages should have been SSG/ISR.
- **Mixing routers and expecting shared layout** — `pages/_app` does not wrap `app/` routes and vice versa; the two trees are separate.
- **Migrating data functions verbatim** instead of rethinking at component granularity — you lose the main benefit if you keep one fat fetch at the top.
- **Using `getInitialProps`** in new code — it disables automatic static optimization globally.

## See Also

- [[app-router-model]]
- [[file-conventions]]
- [[route-handlers]]
- [[server-and-client-components]]
