# Context

Context is React's mechanism for passing a value through the tree without threading props at every level. It is **not** a state manager and not a performance optimization — it is *dependency injection scoped to the render tree*. The crucial internal fact: when a Provider's `value` changes by `Object.is`, **every consumer of that context re-renders**, regardless of `React.memo` between them, because context propagation pierces memoization boundaries. Misunderstanding this is the source of nearly every context performance complaint.

## Mechanics

```js
const ThemeContext = createContext(defaultValue);
// Provider sets the value for its subtree
<ThemeContext value={theme}>{children}</ThemeContext> // React 19: <Context> is its own Provider
// Consumer reads the nearest matching Provider above it
const theme = useContext(ThemeContext);
```

`createContext` produces an object holding a `Provider` (and legacy `Consumer`) plus a current value cell. In **React 19**, `<MyContext>` can be rendered directly as the provider — `<MyContext.Provider>` still works but the bare form is the new idiom (and `Context.Provider` is on the deprecation path).

When React renders a Provider, it pushes the `value` onto a stack during the render pass and pops it after, so the lookup is lexical-by-tree-position: `useContext` reads the value of the **nearest Provider above** the consuming fiber. `defaultValue` is used only when *no* Provider exists above — a frequent gotcha is forgetting the Provider and silently getting the default.

## How Propagation Forces Consumer Re-Renders

This is the part that surprises seniors. Normal updates flow parent→child and stop at `React.memo` / bailout boundaries. **Context updates don't.** When a Provider commits a new `value`, React walks the subtree and, for every fiber that *reads this specific context*, schedules a re-render — even if that fiber sits behind a memoized parent whose props didn't change.

```
<Provider value={obj}>      // obj is a NEW object each render
  <Memoized>                // props unchanged → would normally skip...
    <Consumer/>             // ...but it reads context → re-renders anyway
  </Memoized>
</Provider>
```

The trigger is `Object.is(prevValue, nextValue)`. So the #1 pitfall is an **inline object/array as `value`**: `value={{ user, setUser }}` allocates a new reference every Provider render, so *all* consumers re-render every time the Provider's parent renders, even when `user` didn't change. Fix: memoize the value (`useMemo(() => ({ user, setUser }), [user])`) — see [[memoization]].

## Performance Pitfalls & Context Splitting

Context has **no selector mechanism by default** — a consumer subscribes to the *whole* value, not the slice it uses. If one context carries `{ user, theme, cart }`, a `cart` change re-renders `theme`-only consumers. Mitigations:

- **Split contexts by change frequency / concern:** separate `UserContext`, `ThemeContext`, `DispatchContext`. Consumers only re-render for their own context. This is the primary, idiomatic fix.
- **Separate state from dispatch:** put the rarely-changing `dispatch`/setters in one context and the volatile state in another. Components that only dispatch never re-render on state changes (dispatch identity is stable — [[state-and-reducers]]).
- **Stabilize the value** with `useMemo` so the Provider doesn't churn references.
- **Push state down / lift content up:** wrap `{children}` so children passed as props aren't re-created by the Provider's own re-render.

There is **no built-in `useContextSelector`**; the much-discussed selector pattern is library territory. React deliberately keeps context coarse-grained.

## Context vs External Stores

Context is a *transport* for a value down the tree; it is **not optimized for high-frequency updates**. For app state with many independent slices and frequent writes, an external store (Redux, Zustand, Jotai, a hand-rolled store + `useSyncExternalStore`) is usually better because:

- Stores support **selector-based subscriptions** — components re-render only when their selected slice changes, sidestepping context's all-consumers-re-render behavior.
- Stores live *outside* React, so updates don't require a Provider re-render to propagate.
- Stores integrate with `useSyncExternalStore` to stay tear-free under concurrent rendering ([[concurrent-hooks]]).

The common, correct hybrid: **use context to inject the store instance (DI), and the store's own subscription mechanism for granular reads.** Context transports the store; the store handles the reactivity.

Choose context for: theming, locale, current user, auth, a configured client/SDK instance, dependency injection — things that change rarely and are read widely. Choose an external store for: frequently-updated, finely-sliced application state.

## Senior Mental Model & Misconceptions

- "Context replaces Redux" — only for *low-frequency, widely-read* values. It lacks selectors, devtools, middleware, and granular subscriptions. It re-renders bluntly.
- "`React.memo` will stop context re-renders" — **false.** Context bypasses memoization for its consumers. `memo` only stops *prop-driven* re-renders.
- Context isn't free at the read site, but its main cost is the **fan-out re-render** on value change, not the read itself.
- Reading context still obeys the Rules of Hooks ([[rules-of-hooks]]); but note React 19's `use(Context)` can read context **conditionally** ([[concurrent-hooks]]).
- Think of it as **scoped DI**: a value injected for a subtree. That framing keeps you from abusing it as a global mutable store.

## See Also

- [[hooks-internals]]
- [[memoization]]
- [[state-and-reducers]]
- [[concurrent-hooks]]
- [[rules-of-hooks]]
- [[06-software-design/software-architecture/dependency-injection|Dependency Injection]]
