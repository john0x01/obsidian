# Render And Commit Phases

Every React update moves through two distinct phases. The **render phase** builds a work-in-progress fiber tree describing *what the UI should be* — it is interruptible, may be restarted or thrown away, and **must be pure**. The **commit phase** takes a finished WIP tree and applies it to the actual host (DOM), runs refs and lifecycle/effect callbacks, and is **synchronous and atomic**. Keeping these phases separate is the foundation that makes concurrent rendering possible: you can pause and replay something that touches nothing observable, but you can never pause halfway through mutating the DOM.

## Render Phase

During render, React calls your components, runs hooks, and reconciles children, producing a WIP tree via `beginWork`/`completeWork` (see [[work-loop-and-scheduling]]). Two properties define it:

- **Interruptible / restartable.** A higher-priority update can preempt an in-progress render; React discards the partial WIP tree and starts over. The render phase can therefore execute your function component, `useMemo` factories, and reducers **more than once** before anything commits, in any order relative to other components.
- **Pure.** Given the same props and state, render must produce the same tree and cause no observable side effects. This is not stylistic dogma — it's a correctness requirement that falls directly out of "this code may run twice and be thrown away." Mutating a ref, a module variable, or external state during render means a discarded render leaves corrupting residue, and a replayed render double-applies it. React's `StrictMode` *intentionally double-invokes* render-phase functions in development precisely to surface impurity.

What's safe in render: deriving values, calling other pure functions, reading props/state. What's not: network requests, DOM reads/writes, mutating refs, subscribing to stores imperatively, logging that matters, scheduling timers. All of those belong in the commit phase via effects.

## Commit Phase

Once a WIP tree is fully built and reconciled, React commits it. This phase walks only the fibers flagged with effects (using `subtreeFlags` to skip clean subtrees — see [[fiber-architecture]]) and runs in three sub-steps:

1. **Before mutation** — snapshots (`getSnapshotBeforeUpdate`), and scheduling of passive effects.
2. **Mutation** — the actual DOM operations: insertions (`Placement`), updates, deletions; detaching old refs; running `useLayoutEffect` *cleanups*.
3. **Layout** — after the DOM is mutated, attaching refs and running `useLayoutEffect` callbacks (and `componentDidMount`/`DidUpdate`). These run synchronously *before the browser paints*, so they can read freshly-committed layout.

Between mutation and layout, React flips the root's `current` pointer to the WIP tree — the O(1) double-buffer swap. **Passive effects** (`useEffect`) are *not* run synchronously here; they're scheduled to flush asynchronously after paint, so they don't block the frame.

```text
RENDER (interruptible, pure)          COMMIT (sync, atomic)
  beginWork / completeWork    ──▶   before-mutation │ mutation │ layout
  build WIP tree, may restart        flip current ↑        paint → passive effects
```

## Double Buffering

React maintains two trees: **current** (on screen) and **work-in-progress**. Render mutates only the WIP tree, leaving `current` pristine — so if a render is abandoned, the live UI is untouched and React still has a clean baseline to diff against. Commit's pointer flip makes the WIP tree the new current in one step. This is why a thrown-away concurrent render is cheap and safe: nothing user-visible ever depended on it.

## Why Render Must Be Side-Effect-Free

The single most consequential senior-level takeaway: **the render phase has no guarantees about how many times it runs or whether its output is used.** Concurrent React leans on this hard — time slicing, transitions, Suspense retries, and `StrictMode`'s double-invoke all exploit the freedom to run render speculatively. Code that assumes "render = it happened once and for real" breaks subtly under load (where extra renders actually occur in production) while passing in trivial dev scenarios. Commit, by contrast, runs exactly once per committed update and in a defined order, which is why every observable effect is anchored there.

## Senior Pitfalls

- **`useLayoutEffect` blocks paint.** It runs synchronously in the layout step; heavy work here reintroduces jank that concurrent rendering tried to remove. Reserve it for measuring/mutating layout before the user sees a flicker.
- **Tearing risk lives in render.** Reading mutable external state directly during render can produce inconsistent values across an interrupted render — the reason `useSyncExternalStore` exists (see [[concurrent-rendering]]).
- **State updates during render** are only legal in the narrow "render-phase update" pattern (calling a setter during render to immediately recompute); arbitrary mutation is not.
- **Effects' ordering ≠ render ordering.** Children's effects commit before parents' layout effects; don't assume render order equals effect order.

## See Also

- [[fiber-architecture]]
- [[work-loop-and-scheduling]]
- [[concurrent-rendering]]
- [[suspense]]
- [[10-frameworks-and-stacks/react-native/architecture/rendering|RN Rendering]]
