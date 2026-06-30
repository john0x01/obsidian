# External Stores

An external store is shared mutable state that lives **outside React's render tree** — in a plain module-level object that components subscribe to. Redux, Zustand, Jotai, and Valtio are all variations on this idea. They exist because React's built-in tools (Context, lifted `useState`) broadcast indiscriminately: any change re-renders every consumer. An external store inverts control — components subscribe to *slices* and re-render only when their slice changes — which is the property Context structurally cannot provide.

## useSyncExternalStore: the official primitive

Since React 18, the sanctioned way to read an external store is `useSyncExternalStore`. Every serious store library uses it under the hood. Its signature:

```js
const value = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
```

- `subscribe(callback)` registers `callback` to fire on any store change and returns an unsubscribe function.
- `getSnapshot()` returns the current value React should render. **It must return a referentially stable value when nothing changed** — React compares snapshots with `Object.is`. Returning a fresh object/array each call (`getSnapshot: () => ({...state})`) causes an infinite render loop; this is the single most common misuse.
- `getServerSnapshot()` supplies the value during SSR and the initial hydration pass.

Why a dedicated hook rather than a `useEffect` subscription? Because subscribing in an effect opens a **gap**: between render and effect-commit the external value can change, and you'd render stale data, then tear. `useSyncExternalStore` closes that gap by re-checking the snapshot at the right point in the commit and forcing a synchronous re-render if it drifted.

## Tearing under concurrency

Tearing is the failure this primitive was built to prevent. Concurrent rendering lets React render a tree across multiple yields. If an external mutable value changes *mid-render*, components rendered before the change see the old value and components rendered after see the new one — a single visual frame shows two inconsistent versions of the same source of truth. Internal React state can't tear because React versions it; a plain mutable object can.

`useSyncExternalStore` defeats tearing by **opting that subtree out of concurrent tearing**: when it detects the snapshot changed during a concurrent render, it bails out and re-renders synchronously so the whole tree observes one consistent value. The trade-off is real — reads from external stores can't fully participate in time-slicing — which is part of why atom-based libraries that lean on React's own state for storage are sometimes preferred.

## Selectors and immutability

Two patterns make stores performant and correct:

**Selectors** narrow what a component depends on. `useStore(s => s.user.name)` re-renders only when `name` changes, not on every store mutation. The selector's *return value* is compared (`Object.is` by default), so a selector that builds a new object every call defeats the optimization — you need a shallow-equality comparator (`useSelector(sel, shallowEqual)`) or memoized/atomic selectors (Reselect's `createSelector`) to cache derived shapes.

**Immutability** is what makes change detection cheap. If updates always produce new references for changed branches and reuse references for unchanged ones, equality checks reduce to `prev === next` pointer comparisons — O(1) instead of deep traversal. Redux mandates this; Immer (used by Redux Toolkit) gives you the ergonomics of mutation while producing immutable results via a Proxy draft. Valtio inverts this: you *do* mutate a proxy, and it tracks which properties each component actually read, re-rendering precisely those — immutability is hidden, fine-grained reactivity is the model.

## The Flux/Redux philosophy vs atoms

```
Flux:   Action ──▶ Dispatcher ──▶ Store ──▶ View ──▶ (Action)
                    one-way, single direction, no back-edges
```

Redux is Flux distilled: **one** store, state is read-only, the *only* way to change it is dispatching an action describing what happened, and pure reducers `(state, action) => newState` compute the next state. The payoff is a serializable, replayable, time-travelable event log — the entire history of the app is a list of actions you can inspect, diff, and undo. The cost is ceremony and a single centralized tree that everything funnels through. Redux Toolkit removes most boilerplate but keeps the philosophy.

**Atom-based** stores (Jotai, Recoil) reject the single tree. State is decomposed into many small independent atoms; components subscribe to the specific atoms they use, and *derived atoms* compose into a dependency graph that recomputes only affected nodes. This is bottom-up (compose small pieces) versus Redux's top-down (one tree, slice it). Atoms map naturally onto React's mental model — an atom feels like `useState` you can share — and avoid the selector-equality dance, at the cost of Redux's global-log auditability.

**Zustand** sits between: a single store like Redux but with hook-based selector subscriptions and no reducer/action ceremony — closures mutate via `set`. **Valtio** is the proxy-mutation model above.

## Senior pitfalls

- Reaching for a global store when the state is local, server, or URL state in disguise — server cache especially belongs in a query library, not Redux.
- Unstable snapshots/selectors triggering loops or killing memoization.
- Putting non-serializable values (functions, class instances) in a Redux store, breaking time-travel and persistence.
- Treating Context as a store and then blaming React for re-renders that Context was never designed to suppress.

## See Also

- [[state-architecture]]
- [[rsc-model]]
- [[declarative-ui]]
