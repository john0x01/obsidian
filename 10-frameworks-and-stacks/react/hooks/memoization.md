# Memoization

`useMemo`, `useCallback`, and `React.memo` exist to preserve **referential equality** across renders so that downstream `Object.is` comparisons can short-circuit work. They do nothing for correctness; they are caches with a cost. The senior skill is not "memoize aggressively" but "know exactly which equality check you're feeding and whether the savings beat the bookkeeping." In React 19, the React Compiler shifts most of this work off your hands.

## The Real Problem: Referential Equality

Every render creates new object/array/function literals. JSX props are compared by `Object.is`. So a child that's `React.memo`'d still re-renders if you pass it `onClick={() => ...}` (new function every render) or `style={{...}}` (new object). Memoization hooks let you keep the *same reference* when inputs are unchanged:

- `useMemo(fn, deps)` — caches the **return value** of `fn`; recomputes only when `deps` change (by `Object.is`).
- `useCallback(fn, deps)` — caches the **function itself**; sugar for `useMemo(() => fn, deps)`. Same node shape (`memoizedState = [value, deps]`, see [[hooks-internals]]).
- `React.memo(Component, areEqual?)` — wraps a component so it skips re-render when its **props** are referentially equal (shallow by default, or your custom comparator).

These compose: `React.memo` only pays off if the props you pass are stabilized, which usually requires `useMemo`/`useCallback` upstream. Memoizing a callback while leaving the child unwrapped is pure overhead — nothing consumes the stable reference.

## When Memoization Actually Pays Off

Three legitimate cases:

1. **Genuinely expensive computation** in `useMemo` (large derivations, sorting/filtering big lists) where recompute cost > cache cost.
2. **Stabilizing a reference** that feeds a `React.memo`'d subtree, a `useEffect` dependency array ([[effects]]), or another hook's deps — to prevent cascading re-renders/effect re-runs.
3. **Stable identity for context values** so consumers don't all re-render ([[context]]).

If a value isn't expensive *and* isn't a dependency of something that checks equality, memoizing it is net-negative.

## The Cost Model

Memoization is never free:

- **Allocation:** a hook node, a deps array, the cached value all stay alive — extra memory and GC pressure ([[07-performance-engineering/garbage-collection|Garbage Collection]]).
- **Comparison:** every render runs the `Object.is` loop over `deps`.
- **Cognitive/maintenance:** deps arrays drift out of sync; a wrong dep reintroduces stale values; the cache can mask bugs.
- **It's a cache, not a guarantee:** React may *throw away* a `useMemo` cache (documented behavior — e.g. for memory). Never rely on `useMemo` for correctness or to run something exactly once. `useMemo` is for performance only; if you need a guaranteed-stable mutable value use `useRef` ([[refs]]).

Net rule: memoize when the *measured* cost of recompute/re-render exceeds the standing cost of the cache. "Might be slow" is not a reason; a profiler reading is.

## Common Misuse (Senior-Level)

- **Cargo-cult wrapping** of every callback/value "just in case." This bloats code, adds compares, and usually stabilizes references nothing consumes.
- **`useMemo` for cheap primitives** (`useMemo(() => a + b, [a,b])`) — the compare costs more than the add.
- **Broken dep chains:** memoizing the inner callback but recreating the object it closes over → cache never hits.
- **Inline objects in deps:** `useMemo(..., [{x}])` — new object every render defeats the cache.
- **Relying on cache persistence** for one-time work — use lazy init in [[state-and-reducers]] or a ref instead.
- **Premature `React.memo`** on cheap components: the shallow prop compare can cost more than just re-rendering a few divs. Reconciliation is already fast; memoize subtrees that are *deep* or *expensive*, not leaves.

## How The React Compiler Changes Everything

The **React Compiler** (formerly "React Forget"), now a stable-track tool in the React 19 era, performs build-time auto-memoization. It analyzes your component, infers the dependency graph of every value and JSX expression, and emits memoization (a hidden per-render cache, conceptually slots holding `[deps, value]`) so that values/callbacks/JSX are recomputed only when their actual inputs change — at a *finer granularity* than hand-written `useMemo` (it can memoize sub-expressions, not just whole values).

Implications for how you write code:

- Most manual `useMemo`/`useCallback`/`React.memo` becomes **unnecessary** and is essentially what the compiler does for you, more correctly.
- The compiler is **safe only if you follow the Rules of Hooks and write idiomatic, side-effect-free render code** ([[rules-of-hooks]]); it *bails out* of components it can't prove safe (e.g. ones that mutate during render), leaving them un-optimized rather than wrong.
- You still occasionally hand-memoize at integration boundaries (e.g. props handed to non-compiled or third-party `memo` components, or expensive non-React work) — but the default posture flips from "memoize defensively" to "write clean code, let the compiler optimize."

## Mental Model

Memoization is **referential-equality engineering**, not speed magic. Always trace the chain: *which `Object.is` check am I trying to pass, and what re-render/effect/recompute does passing it actually save?* If you can't name the consumer of the stable reference, delete the memo. And in React 19, prefer enabling the compiler over scattering manual caches.

## See Also

- [[hooks-internals]]
- [[state-and-reducers]]
- [[context]]
- [[refs]]
- [[07-performance-engineering/memoization|Memoization (Performance)]]
- [[07-performance-engineering/garbage-collection|Garbage Collection]]
