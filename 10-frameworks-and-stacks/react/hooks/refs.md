# Refs

A ref is a **mutable box that survives renders without causing them**. `useRef(init)` returns the *same* object — `{ current: init }` — on every render of a component instance, and mutating `.current` is invisible to React's render cycle. That single property (persistent identity, render-transparent mutation) is what makes refs the canonical escape hatch: for everything React's declarative model deliberately can't express.

## The Box Model

```js
const ref = useRef(0);
// ref.current is the only mutable slot; ref itself is stable forever
```

Internally the ref *is* the hook node's `memoizedState`: `{ current }` (see [[hooks-internals]]). React creates it on mount and returns the identical reference on every subsequent render. Reading or writing `.current` does not enqueue an update, does not run a compare, does not re-render. The box persists because the fiber persists; the *contents* are entirely under your control.

Two distinct uses share this primitive:

1. **Instance variables** — values that persist across renders but shouldn't trigger them: timer IDs, previous values, latest-value caches, mutable flags, third-party library instances.
2. **DOM access** — `<input ref={inputRef} />`; after commit, `inputRef.current` is the DOM node. React sets it during the commit phase, before layout effects.

## Refs vs State — The Core Distinction

|                     | `useState` | `useRef` |
|---------------------|------------|----------|
| Triggers re-render  | yes        | no       |
| Value for *this render* | snapshot (immutable binding) | always reads latest `.current` |
| Mutate in place     | no (use setter, new ref) | yes (`.current = x`) |
| Read during render  | always safe | discouraged (not render-pure) |

The decision rule: **does the UI need to react to this value?** Yes → state. No (it's bookkeeping the render doesn't depend on) → ref. A ref read during render breaks purity because two renders with identical props/state can read different `.current` values — React's contract assumes render is a pure function of props+state. Mutating `.current` *during* render is likewise unsafe (except the lazy-init-once idiom). Confine ref mutation to event handlers and effects.

Refs also defeat the stale-closure problem in [[effects]]: keep a `latestRef.current = value` updated each render, and an effect/callback can read the freshest value without listing it as a dependency (the "latest ref" pattern) — used carefully, it decouples "read latest" from "re-subscribe on change."

## Callback Refs

Instead of a ref object, pass a **function** to `ref`. React calls it with the node on attach and with `null` on detach:

```jsx
<div ref={node => { if (node) observe(node); }} />
```

Callback refs run during commit and are strictly more powerful than object refs: you can measure immediately, attach observers, or manage a *dynamic list* of nodes (a `Map` keyed by id). Caveat: an inline callback ref has a new identity each render, so React detaches (`null`) and re-attaches on every render. Stabilize with `useCallback` if that matters. **React 19** adds **cleanup functions for callback refs** — return a function from the callback and React calls it on detach instead of invoking with `null`, mirroring effect cleanup.

## Ref-As-A-Prop in React 19 (forwardRef Deprecation Direction)

Historically `ref` was a reserved prop you couldn't receive normally; you needed `forwardRef`. **React 19 lets function components receive `ref` as an ordinary prop**:

```jsx
function Input({ placeholder, ref }) {     // ref is just a prop now
  return <input ref={ref} placeholder={placeholder} />;
}
```

`forwardRef` still works but is on the **deprecation path** — the direction is plain props, with a codemod to migrate. This removes a long-standing wart (the wrapper component, the awkward second argument) and makes ref-passing compose like any other prop.

## useImperativeHandle

When a parent holds a ref to your component, by default it gets the DOM node. `useImperativeHandle(ref, () => ({ ... }), deps)` lets you expose a **curated imperative API** instead, hiding the raw node:

```js
useImperativeHandle(ref, () => ({
  focus: () => inputRef.current.focus(),
  scrollIntoView: () => inputRef.current.scrollIntoView(),
}), []);
```

Use it to expose intent (`focus`, `play`, `reset`) rather than the full DOM surface — it narrows the imperative contract. It's a deliberate escape hatch; reach for it only when declarative props genuinely can't express the interaction (focus management, media playback, animations, third-party imperative widgets).

## The Escape-Hatch Philosophy

Refs are React's pressure-release valve: the declarative model covers ~95% of UI, and refs handle the imperative remainder without polluting the declarative parts. The philosophy:

- **Stay declarative by default.** Reaching for a ref to "force" something React should derive from state is usually a smell — you're fighting the model.
- **Refs are for things outside React's data flow:** the DOM, timers, observers, focus, scroll position, third-party libs, latest-value caches.
- **Don't read/write `.current` during render.** Confine mutation to effects and handlers to keep render pure.
- **Prefer state until you can articulate why the UI must *not* react** to a value. The burden of proof is on choosing a ref.

## Senior Pitfalls

- Storing derived UI data in a ref to "avoid re-renders" — the UI then goes stale; that value wanted state.
- Forgetting inline callback refs re-run every render (detach/reattach churn); memoize when attaching observers.
- Assuming `ref.current` is populated during render — it's set in commit, available in effects, `null` on the first render before mount.
- Mutating a ref expecting the screen to update — it never will; that's the whole point.

## See Also

- [[hooks-internals]]
- [[state-and-reducers]]
- [[effects]]
- [[concurrent-hooks]]
- [[01-programming-foundations/paradigms/declarative-programming|Declarative Programming]]
