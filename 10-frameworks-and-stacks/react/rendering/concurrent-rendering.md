# Concurrent Rendering

**Concurrent rendering** is React's ability to prepare multiple versions of the UI at the same time, pausing, resuming, interleaving, or abandoning render work according to priority — all on a single thread. It is not a feature you call; it's a *capability* that the fiber work loop and the lane scheduler unlock (see [[work-loop-and-scheduling]] and [[lanes-and-priorities]]), exposed through an opt-in surface of features. The philosophical shift: rendering stops being an all-or-nothing synchronous transaction and becomes an interruptible negotiation between your UI work and the browser's responsibilities.

## What "Concurrent" Actually Means Here

There is no parallelism — no extra threads, no Web Workers. "Concurrent" means React can have an in-progress render that it *holds in memory* (the work-in-progress tree) while choosing to do something more urgent first, then come back. Because each unit of work is a heap-allocated fiber and the loop checks `shouldYield()` between units, React can:

- Interrupt a low-priority render to handle a click, then resume the render.
- Render a not-yet-visible state in the background while the current UI stays interactive.
- Throw away a render that's now stale and start a fresher one.

The legacy synchronous reconciler could do none of this — once it started, it blocked until done.

## What Concurrent Features Unlock

Concurrency is the substrate for React 19's user-facing primitives:

- **Transitions** (`useTransition`/`startTransition`) — mark updates non-urgent so they render at a lower lane and yield to input (see [[transitions]]).
- **`useDeferredValue`** — render a stale value at high priority and the fresh one in the background.
- **Suspense** — suspend a subtree and show a fallback without blocking the rest of the tree; coordinate reveals with transitions (see [[suspense]]).
- **Streaming SSR + selective hydration** — server-stream HTML and hydrate interactive regions out of order, prioritizing what the user touches.

None of these are achievable under a render model that can't pause.

## Tearing And `useSyncExternalStore`

Concurrency introduces one genuinely new hazard: **tearing**. Because a render can be interrupted and resumed across time slices, an external mutable source (a Redux store, a global, a `useRef`-held value) might be read by some components *before* a mutation and others *after* it within the **same render pass**. The committed UI would then show two values for one source — visually torn, internally inconsistent. The synchronous reconciler was immune simply because it read everything in one uninterruptible burst.

`useSyncExternalStore` is the sanctioned fix. It forces React to:

- Read the snapshot in a way that's checked for consistency; if the store changed mid-render, React detects it and **synchronously re-renders** to a consistent snapshot rather than committing a torn one.
- Drop external stores out of the concurrent fast path *when needed* to preserve consistency.

```js
const value = useSyncExternalStore(
  store.subscribe,      // (cb) => unsubscribe
  store.getSnapshot,    // must return a stable, comparable snapshot
  store.getServerSnapshot, // optional, for SSR/hydration parity
);
```

This is why every serious external-store library (Redux, Zustand, Jotai) routes through it. The senior takeaway: **any state outside React's own state must use `useSyncExternalStore` to be concurrent-safe.** `getSnapshot` must return referentially stable values for unchanged data, or you'll trigger infinite re-renders.

## The Opt-In Model

A deliberate design choice: turning on `createRoot` enables the concurrent *renderer*, but concurrent *behavior* (interruption, deferral) only kicks in for updates that are actually marked low-priority. An ordinary `setState` still renders synchronously-ish at default priority. This gradual, opt-in model preserved backward compatibility — existing apps got the new renderer without behavior changes, and teams adopt concurrency feature-by-feature. The cost is conceptual: the same `setState` behaves differently depending on whether it's wrapped in a transition.

## Responsiveness Philosophy

The animating idea is that **not all updates are equally urgent**, and the framework — not the developer hand-rolling debounces — should be able to keep the UI responsive by reordering work. Typing into a filter box is urgent; re-rendering the 10,000-row results list is not. Pre-concurrency, both happened in the same blocking render, so typing stuttered. Concurrent React lets the urgent keystroke interrupt the expensive list render. You express *intent* ("this is a transition") and React handles the scheduling.

## Senior Pitfalls

- **Concurrency ≠ free performance.** It reduces *blocking*, not total work. An unmemoized expensive component is still expensive; transitions just stop it from blocking input.
- **Double-rendering surfaces here.** Speculative/abandoned renders mean impure render code breaks under concurrency (see [[render-and-commit-phases]]); `StrictMode` double-invoke is the early-warning system.
- **Tearing is silent.** It only manifests under interruption + concurrent stores, so it rarely shows in dev. Treat `useSyncExternalStore` as mandatory for external state, not optional.
- **`useDeferredValue` shows stale UI by design.** If you don't visually signal staleness, users see lag they can't explain.

## See Also

- [[work-loop-and-scheduling]]
- [[lanes-and-priorities]]
- [[transitions]]
- [[suspense]]
- [[render-and-commit-phases]]
- [[10-frameworks-and-stacks/react-native/architecture/new-vs-old-architecture|RN New Architecture]]
- [[concurrent-hooks]] — the user-facing API
