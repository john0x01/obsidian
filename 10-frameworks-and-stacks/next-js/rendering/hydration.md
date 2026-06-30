# Hydration

Hydration is the process of taking the static HTML the server already produced and **attaching React to it on the client** — reconstructing the component tree, wiring up event listeners, and restoring state — *without re-creating the DOM*. The server gives you pixels fast; hydration is what makes those pixels interactive. Its cost, its failure modes, and the way RSC shrinks it are some of the highest-leverage things a senior Next.js engineer can understand, because hydration is where "fast first paint" and "slow time-to-interactive" diverge.

## What Hydration Actually Does

`renderToString`/streaming gives the browser real markup. On the client, `hydrateRoot` (which Next calls for you) walks the existing DOM and the rendered React tree *in parallel*, and instead of producing new DOM nodes it **adopts** the existing ones — claiming each node, attaching the synthetic event system, and building the fiber tree and hooks state behind it. The expensive part is not painting (already done) but **executing all the component code** to know which listeners and state attach where.

```
SSR HTML (no listeners)  ──hydrate──►  same DOM, now React-controlled
   visible, inert                         interactive
```

A critical consequence: between first paint and hydration completion, the page *looks* ready but isn't — buttons don't respond, inputs may be uncontrolled. This gap is the real meaning of a poor TTI despite a great FCP.

## Mismatch Causes And Debugging

`hydrateRoot` assumes the client's first render produces **byte-identical** structure to the server HTML. When it doesn't, React logs a hydration error and, for the mismatched subtree, discards the server DOM and re-renders on the client — slow and visually janky. Classic causes:

- **Non-deterministic render** — `Date.now()`, `Math.random()`, `new Date().toLocaleString()` (timezone/locale differs server↔client), `window`/`localStorage` read during render.
- **Invalid HTML nesting** the browser auto-corrects (`<p>` inside `<p>`, `<div>` inside `<table>`), so the parsed DOM differs from what React rendered.
- **Browser-extension or third-party DOM mutation** before hydration.
- **Branching on `typeof window`** during render, producing different trees on each side.

Debugging: read the error — React 18+/19 prints the specific diverging element. For intentional, unavoidable differences (e.g., timestamps), use `suppressHydrationWarning` on that element, or defer the client-only value to a `useEffect` so the first client render matches the server and the real value paints after. For "this can only render on the client" components, gate with a mounted flag or a `dynamic(() => ..., { ssr: false })` import — accepting that you lose SSR for that subtree.

## Progressive And Selective Hydration

React 18 made hydration **non-blocking and out-of-order**, gated by Suspense:

- **Progressive hydration** — boundaries hydrate independently as their HTML streams in, rather than the whole page hydrating in one synchronous, main-thread-blocking pass.
- **Selective hydration** — React *prioritizes* the boundary the user interacts with. Click a not-yet-hydrated island and React hydrates it ahead of the queue so the event isn't lost. It can also yield between boundaries so hydration doesn't block input.

This is why the Suspense boundaries you place for [[streaming-ssr]] pay off twice: they're both streaming units *and* hydration units.

## How RSC Reduces Hydration

The deepest lever: **Server Components don't hydrate at all.** They render only on the server; their output is serialized into the RSC payload and they ship **zero client JS**. There is no component code to download, parse, or execute for them on the client — nothing to attach. Only `'use client'` components hydrate.

```
Server Component  → HTML + RSC payload, no client bundle, no hydration
Client Component  → HTML + JS bundle, hydrated on the client
```

So in the App Router the hydration cost is proportional to your **Client Component surface area**, not your whole tree. Pushing the `'use client'` boundary *down* — keeping data-fetching and static structure in Server Components and isolating interactivity into small leaf islands — directly shrinks the JS that must be shipped and hydrated. This is the single biggest TTI lever the architecture gives you, and the main reason RSC exists from a performance standpoint.

## The Cost Of Hydration

Hydration is pure overhead from the user's perspective: the page was already visible; this work only buys interactivity. Its cost scales with bytes of client JS to download, parse/compile time, and the synchronous render pass to rebuild fibers and hooks. On low-end mobile this is often the dominant slice of the "white screen → usable" budget. The pathological case — server-render a huge tree, then hydrate all of it — pays *twice* for content that may never need interactivity. RSC's answer is to simply not hydrate what doesn't need it.

## Senior Pitfalls And Philosophy

- **A mismatch is a correctness bug, not a cosmetic warning.** It can silently re-render to *different* content than was painted, and it disables progressive hydration for that subtree.
- **`suppressHydrationWarning` is surgical, not a mute button.** It suppresses one element's text/attr diff; it does not fix structural mismatches.
- **`'use client'` is a *boundary*, not a *file flag*.** Everything imported into a client module joins the client bundle, so a stray directive high in the tree drags huge subtrees into hydration. Audit where the boundary sits.
- **First client render must equal the server render — full stop.** Want client-only data? Render the server-safe version first, then update in `useEffect`.
- **`ssr: false` trades SEO/first-paint for zero mismatch risk** — legitimate for genuinely client-only widgets, but make it deliberately.

Philosophy: hydration is the tax for making server HTML interactive, and the architecture's job is to *minimize what is taxed*. The App Router's whole shape — Server Components by default, Client Components as small islands, Suspense as hydration units — is an extended argument that most of a page should never hydrate at all.

## See Also
- [[rendering-strategies]]
- [[streaming-ssr]]
- [[static-and-isr]]
- [[partial-prerendering]]
- [[10-frameworks-and-stacks/react/fiber/fiber-architecture|Fiber Architecture]]
