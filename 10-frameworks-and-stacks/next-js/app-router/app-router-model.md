# The App Router Mental Model

The App Router is not "the Pages Router with a new folder name." It is a re-architecture around a single thesis: **the server is the default execution environment, and the page is composed from a tree of independently-rendered segments rather than a single component.** Understanding it well means thinking in terms of a *route tree* of nested layouts and React Server Components, not a flat map of URL→component.

## The Route Tree, Not the Route Map

In the Pages Router, `pages/blog/[slug].tsx` is one file that *is* the page. The mental model is a dictionary: URL maps to one default-exported component, plus a sidecar `getServerSideProps`. The App Router instead treats `app/` as a filesystem-encoded tree where each **segment** (each folder) contributes UI by colocating special files (`layout`, `page`, `loading`, `error`, `template`). The rendered output is the *composition* of every segment from root to leaf:

```
app/layout.tsx          → RootLayout (wraps everything, owns <html>/<body>)
  app/dashboard/layout.tsx → DashboardLayout (persists across child nav)
    app/dashboard/page.tsx → the actual leaf page
```

The router renders this as nested React: `<RootLayout><DashboardLayout><Page/></DashboardLayout></RootLayout>`. The crucial property: **layouts persist across navigation.** Navigating from `/dashboard/a` to `/dashboard/b` re-renders the page leaf but *does not* unmount `DashboardLayout` — its state, scroll position, and any client components inside it survive. This is implemented via the React Server Component payload being diffed per-segment, not via a full page swap.

## Server-First / RSC by Default

Every component under `app/` is a **Server Component unless it (or an ancestor in the module graph) is marked `'use client'`.** This inverts the historical default. Server Components:

- Run only on the server (at request time or build time), never ship their code to the browser.
- Can be `async` and `await` data directly — no `useEffect`/`useState` data-fetching dance.
- Can read secrets, hit the database, and access the filesystem because their source never reaches the client.
- Cannot use hooks, browser APIs, or event handlers — they render once to a serialized description.

The payoff: large dependencies (markdown parsers, date libraries, ORM clients) used only for rendering stay on the server and contribute **zero** to the client bundle. The mental shift for a Pages-era engineer is to stop reaching for `useEffect`+`fetch` and instead `await` in the component body. See [[server-and-client-components]] for where the boundary actually falls.

## Colocation as a First-Class Idea

Because only `page.tsx` and `route.ts` are publicly routable, you can freely colocate non-routable files inside a segment folder — `components/`, `utils.ts`, tests, a `_private` folder (the `_` prefix opts a folder out of routing entirely). The route tree and the source organization finally coincide. This is a deliberate philosophy: the thing that owns a feature should live *with* that feature, not in a parallel `components/` shadow hierarchy far away.

## What Problems It Actually Solves

- **Layout waterfalls and re-mounting.** Pages Router had no native nested-layout primitive (`_app` was global; per-page layouts were a userland `getLayout` hack). Layouts were re-evaluated and lost state on navigation. Nested layouts fix this structurally.
- **Client-bundle bloat.** RSC keeps rendering-only code off the wire.
- **Sequential data waterfalls.** Streaming + `loading.tsx` (a Suspense boundary) let the shell paint instantly while slow data streams in, instead of blocking the whole route on the slowest fetch.
- **Per-route data lifecycle.** `getServerSideProps` was page-level and all-or-nothing; the App Router fetches at the granularity of each component, with caching/revalidation expressed per fetch.

## Senior Pitfalls and Misconceptions

- **"RSC means no JavaScript ships."** False. Client components still hydrate normally; RSC only removes *server-only* component code. The framework runtime, router, and all `'use client'` subtrees still ship.
- **"Server Components re-render like client components."** They don't re-render in the browser at all. A Server Component "re-renders" only when the server is asked to produce a new RSC payload for that segment (navigation, router refresh, revalidation).
- **Treating `layout.tsx` as a place to fetch per-page data.** Layouts don't re-run on sub-route navigation, so data that must change per child page belongs in the child, not the layout — otherwise it goes stale or forces you to fight the persistence model.
- **Assuming request context flows like SSR globals.** There is no implicit `req`/`res`. Request data comes through typed async APIs (`cookies()`, `headers()`, `params`), and touching them marks a segment dynamic.

The honest summary: the App Router asks you to model your UI as a *composable tree rendered mostly on the server*, with the client boundary as an explicit, intentional choice — not the ambient default.

## See Also

- [[file-conventions]]
- [[server-and-client-components]]
- [[advanced-routing]]
- [[pages-router]]
- [[10-frameworks-and-stacks/react/fiber/fiber-architecture|Fiber Architecture]]
- [[10-frameworks-and-stacks/react/philosophy/declarative-ui|Declarative UI]]
