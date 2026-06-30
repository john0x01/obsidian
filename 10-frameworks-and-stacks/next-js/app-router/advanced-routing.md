# Advanced Routing

Beyond mapping folders to URLs, the App Router has four routing primitives that decouple the *URL structure* from the *organization* and *rendering* of your route tree: route groups, dynamic/catch-all segments, parallel routes, and intercepting routes. The unifying idea is that **the folder structure no longer has to match the URL one-to-one** — you can group, slot, and intercept. These are the features that make non-trivial layouts (dashboards, modals, tabbed views) expressible declaratively.

## Dynamic and Catch-All Segments

The foundation. A folder named with brackets becomes a parameter:

- **`[id]`** — a single dynamic segment. `app/blog/[slug]/page.tsx` matches `/blog/hello`; `params.slug === 'hello'`.
- **`[...slug]`** — catch-all. Matches `/docs/a/b/c`, giving `params.slug === ['a','b','c']`. Requires at least one segment.
- **`[[...slug]]`** — optional catch-all. *Also* matches the parent `/docs` with `params.slug === undefined`.

Pair these with `generateStaticParams()` to pre-render known paths at build time, and `dynamicParams` (default `true`) to decide whether unknown params render on-demand or 404. In Next 15, `params` is a **Promise** — `const { slug } = await params`.

## Route Groups — `(folder)`

A folder wrapped in parentheses **organizes without affecting the URL.** `app/(marketing)/about/page.tsx` serves `/about`, not `/marketing/about`. The parentheses segment is stripped from the path. Why this matters:

- **Multiple root layouts.** Put `(marketing)/layout.tsx` and `(app)/layout.tsx` side by side — two distinct top-level layouts (e.g. a marketing site and a logged-in app) with completely different chrome, sharing no layout, without any URL prefix.
- **Scoping a layout to some siblings but not others** without forcing them under a URL segment.

```
app/
  (marketing)/layout.tsx    → wraps /about, /pricing
    about/page.tsx          → /about
  (app)/layout.tsx          → wraps /dashboard
    dashboard/page.tsx      → /dashboard
```

## Parallel Routes — `@slot`

A folder prefixed with `@` defines a **named slot** that renders *simultaneously* alongside `children` in the same layout. The slot is passed to the layout as a prop named after the folder:

```tsx
// app/dashboard/layout.tsx
export default function Layout({ children, team, analytics }) {
  return <>{children}{team}{analytics}</>   // @team and @analytics slots
}
```

This lets one layout render **multiple independent subtrees, each with its own loading/error state and its own navigation/URL state.** Real use-cases: a dashboard where `@analytics` and `@team` load and fail independently; conditional rendering of different slots based on auth (render `@login` vs `@dashboard` from the same layout). The companion file **`default.tsx`** provides the fallback a slot renders when the current URL doesn't otherwise specify content for it — critical on hard navigation/refresh, where unmatched slots would otherwise 404.

## Intercepting Routes — `(.)`, `(..)`, `(...)`

Intercepting routes let a route **render *different* UI depending on how you arrived at it** — specifically, to show a route's content inside the *current* layout (as an overlay/modal) on soft client navigation, while still serving the full standalone page on hard navigation or refresh. The convention mirrors relative paths:

- `(.)` — intercept a sibling segment (same level).
- `(..)` — intercept one level up.
- `(..)(..)` — two levels up.
- `(...)` — intercept from the app root.

### The Modal Pattern (the canonical use)

Combining **parallel + intercepting routes** produces the "photo modal" / "GitHub-style" pattern: clicking a thumbnail opens the item in a modal over the feed, but visiting/refreshing the URL directly shows the full page.

```
app/
  @modal/
    (.)photo/[id]/page.tsx   ← intercepts /photo/[id] on soft nav → renders in modal slot
    default.tsx              ← renders nothing when no modal active
  photo/[id]/page.tsx        ← the real, full page (hard nav / refresh / share)
  page.tsx                   ← the feed with <Link href="/photo/123">
  layout.tsx                 ← renders {children}{modal}
```

On client-side `<Link>` navigation, the router intercepts and fills the `@modal` slot with the intercepted page, layering it over the feed. On a fresh request to `/photo/123`, no interception happens and the real page renders full-screen. The URL is shareable either way — the magic is that the *same URL* yields two presentations based on navigation context.

## Senior Pitfalls

- **Forgetting `default.tsx`** for parallel slots → unexpected 404s on refresh, because unmatched slots have no fallback.
- **Expecting interception on hard navigation.** Interception only fires on *soft* (client) navigation; a refresh always serves the real route. That asymmetry is the feature, not a bug — but it surprises people testing by reloading.
- **Overusing route groups for layout** when a normal nested layout suffices — groups are for URL decoupling, not a styling tool.
- **Catch-all vs optional catch-all confusion** — `[...x]` does *not* match the bare parent path; `[[...x]]` does.
- **Assuming slots share state automatically** — each parallel slot is an independent subtree with its own boundaries; coordination is explicit.

## See Also

- [[file-conventions]]
- [[app-router-model]]
- [[server-and-client-components]]
- [[route-handlers]]
