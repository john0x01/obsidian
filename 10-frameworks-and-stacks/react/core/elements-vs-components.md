# Elements vs Components

Three distinct things are routinely conflated under the word "component": **elements** (immutable plain-object descriptions of what should appear), **components** (functions that *return* elements), and **fibers/instances** (the mutable internal bookkeeping React keeps for a mounted element). Keeping them separate is the single most clarifying mental model for understanding why React re-renders, when state persists, and what `key` actually controls.

## Elements: immutable descriptions

A React element is a frozen plain object — *not* a DOM node, *not* a component, *not* "on screen." It is a value describing a node:

```js
{ $$typeof: Symbol.for('react.element'), // legacy; see note below
  type: 'div' | Counter,                  // tag string or component function
  key: null,
  props: { className: 'x', children: [...] },
  // ref now lives in props (React 19); previously a top-level field
}
```

Creating an element does nothing visible — it's `createElement('div', {...}, children)` returning data. Elements are **immutable**: once created you cannot change `type` or `props`. To "change" the UI you produce a *new* element tree and let React diff it. An element tree is a cheap, serializable snapshot of intended UI for one render. (In React 19 the internal brand moved toward `Symbol.for('react.transitional.element')` and `ref` is read from `props` rather than a special top-level field — but the element-as-immutable-description concept is unchanged.)

## Components: functions producing elements

A component is a function (`(props) => element-tree`). It is a *factory* for descriptions, invoked by React, not by you. `<Counter n={3} />` does **not** call `Counter(...)`; it creates an element whose `type` is the `Counter` function and whose `props` is `{ n: 3 }`. React calls `Counter` later, during render, when it decides to. This indirection is essential: it's what lets React choose *whether*, *when*, and *how often* to invoke your function — for diffing, for bailouts, for discarding interrupted concurrent renders.

The distinction between `<Counter/>` (an element) and `Counter(props)` (a direct call) is load-bearing. Calling a component as a plain function inlines its output into the parent's render with no element boundary, so it gets no own fiber, no own hooks identity, no reconciliation slot — a classic source of "hooks order changed" crashes.

## Instances / fibers: the mutable runtime

Elements are stateless and thrown away each render. Where does `useState` survive? In the **fiber** — React's internal mutable record for each mounted position in the tree. A fiber holds the hook list, the memoized props, references to DOM nodes, effect lists, and pointers to its child/sibling/parent fibers and to its alternate (the work-in-progress copy). The fiber is the long-lived "instance"; the element is the per-render description that gets reconciled *against* that fiber.

```
Element (per render)      Fiber (persists across renders)
  { type: Counter,   <-->   { type: Counter, memoizedState: <hooks>,
    props: {n:3} }            stateNode, child, sibling, alternate }
```

Reconciliation is the act of matching the *new* element to the *existing* fiber at that slot.

## Element identity and referential reuse

Because elements are values, React can use **referential equality** as a fast-path. If a parent re-renders but passes back the *same element object* for a child (e.g. an element received via `props.children`, or one hoisted to a constant), React can often skip re-rendering that child entirely — the description provably didn't change. This is precisely why passing JSX *through* `children` is a performance pattern: the parent that re-renders didn't create those child elements, so they keep their identity. The React Compiler (React 19) automates much of this element/value memoization that you used to hand-write with `useMemo`/`memo`.

Conversely, *type identity* drives the most important reconciliation decision: comparing the new element's `type` to the fiber's `type`.

- Same `type` at the same slot → React **updates** the existing fiber (state preserved).
- Different `type` → React **unmounts** the old fiber subtree and **mounts** a new one (state destroyed).

This is why defining a component *inside* another component is a bug: each parent render creates a *new function identity* for the inner component, so its `type` differs every time, so React remounts it every render — losing its state and thrashing the DOM.

## Senior pitfalls and misconceptions

- **"Rendering = touching the DOM."** No. "Render" = calling components to produce elements. DOM mutation is a *separate* commit phase that may touch far less (or nothing) after the diff.
- **`key` is about element identity within a sibling list**, not a DOM id. It tells reconciliation which *new* element corresponds to which *existing* fiber, so state follows the right item across reorders. Index keys break this on reorder/insert.
- **Memoizing a component without stable element/props identity does nothing.** `memo` compares props; if the parent recreates an inline object/array/element prop each render, the comparison always fails.
- **An element is not "the rendered output."** It's the input to reconciliation. Confusing the two makes performance work feel like guesswork.

## See Also

- [[virtual-dom]]
- [[reconciliation]]
- [[jsx]]
- [[composition-model]]
- [[10-frameworks-and-stacks/react-native/architecture/rendering|RN Rendering]]
