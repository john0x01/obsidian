# Suspense

**Suspense** lets a component declaratively say "I'm not ready to render yet," and lets an ancestor `<Suspense>` boundary show a fallback while it waits. The mechanism is deliberately humble: a component **throws a promise** (or, in React 19, calls `use(promise)`) during render; React catches it, finds the nearest boundary, renders that boundary's `fallback`, and retries the subtree when the promise settles. This turns asynchronous readiness — data, code, anything — into ordinary control flow that composes through the tree.

## The Throw-A-Promise Mechanism

During the render phase, when a component needs something not yet available, it throws a thenable. React's reconciler, walking the fiber tree (see [[work-loop-and-scheduling]]), catches the throw, marks the fiber as suspended, and unwinds to the nearest `<Suspense>` ancestor, committing its `fallback` instead of the suspended children. It attaches a `.then` to the thrown promise; when it resolves, React schedules a **retry** (on `RetryLanes`) to re-render the subtree, which this time reads the now-available value and renders the real content.

`use` is the React 19 ergonomic over this. `use(promise)` reads a promise's value during render: if pending, it suspends; if resolved, it returns the value; if rejected, it throws to the nearest error boundary. Unlike hooks, `use` may be called conditionally and inside loops, because it's a reader, not a stateful hook tied to call order.

```jsx
function Profile({ userPromise }) {
  const user = use(userPromise); // suspends until resolved
  return <h1>{user.name}</h1>;
}

<Suspense fallback={<Spinner />}>
  <Profile userPromise={fetchUser(id)} />
</Suspense>
```

## Boundaries And Fallbacks

A `<Suspense>` boundary is a catch point for "not ready," analogous to how an error boundary is a catch point for "threw an error." Placement is a design decision: a boundary high in the tree means a large fallback region and a coarse loading state; nested boundaries give **progressive reveal**, where outer content paints while inner regions still load. The fallback is shown only for the suspended subtree, not the whole app — sibling boundaries are independent.

## Coordination With Transitions

This is the subtle part senior engineers miss. When new content suspends, the *default* behavior depends on context:

- If a **freshly mounted** boundary suspends, React shows its fallback immediately.
- If an **already-visible** boundary would be replaced by content that suspends, you usually *don't* want the existing UI to vanish into a spinner. Wrapping the triggering update in a **transition** (`startTransition`) tells React to **keep showing the old content** while the new content loads in the background, rather than reverting to the fallback. `isPending` lets you dim or signal the in-flight state (see [[transitions]]).

So Suspense + transitions together implement the "keep the current screen, swap when ready" UX — the same machinery that powers route transitions without flash-of-spinner. This coordination is only possible because concurrent rendering can prepare the new tree off-screen (see [[concurrent-rendering]]).

## Streaming SSR And Selective Hydration

On the server, Suspense enables **streaming SSR**: React flushes HTML for the parts of the page that are ready, emits the boundary's fallback HTML inline for the parts that aren't, and then — as server data resolves — streams additional HTML chunks plus a tiny inline script that swaps the fallback for the real content in place. The user sees a meaningful shell fast (better TTFB/FCP) without waiting for the slowest data.

On the client, this pairs with **selective hydration**: React hydrates Suspense boundaries independently and out of order, and crucially **prioritizes the boundary the user just interacted with**. If a user clicks a not-yet-hydrated region, React hydrates that region first, then continues with the rest. The page becomes interactive piecewise rather than in one blocking pass.

## Data Fetching With Suspense

Suspense is a *coordination primitive*, not a fetching library. Something must throw the promise. In React 19 + Server Components, the framework (Next.js App Router, etc.) wires this so an `async` Server Component or a promise passed to `use` integrates naturally. A critical rule: **the promise must be created outside render or be stable/cached**, because render can run multiple times. Creating a fresh `fetch()` promise inline on every render causes an infinite suspend-retry loop — each render throws a brand-new pending promise. This is why ad-hoc Suspense data fetching needs a cache (the framework's or a library's); raw `fetch` in a component body does not work.

## Senior Pitfalls

- **Inline promises loop forever.** Memoize/cache the promise; never `use(fetch(...))` created during render.
- **Fallback flicker.** Without a transition, replacing visible content with suspending content flashes the fallback; wrap the update in `startTransition`.
- **Boundary granularity is a UX lever.** Too coarse → whole sections blink; too fine → spinner confetti. Place boundaries around meaningful loading units.
- **Errors vs. suspense are different catches.** A rejected promise routes to an **error boundary**, not the Suspense fallback — you typically need both.

## See Also

- [[concurrent-rendering]]
- [[transitions]]
- [[render-and-commit-phases]]
- [[work-loop-and-scheduling]]
- [[10-frameworks-and-stacks/react-native/architecture/new-vs-old-architecture|RN New Architecture]]
- [[10-frameworks-and-stacks/next-js/rendering/streaming-ssr|Streaming SSR (Next)]] — Suspense-driven streaming
