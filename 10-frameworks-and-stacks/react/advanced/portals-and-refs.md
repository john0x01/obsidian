# Portals And Refs

Portals and refs are React's two principal **escape hatches** — the places where the declarative `UI = f(state)` model deliberately leaks to let you reach the real DOM. Portals break the assumption that a component's DOM output must nest inside its parent's DOM; refs break the assumption that you never touch nodes directly. Both are sanctioned, both are narrow, and using them well means knowing exactly where the abstraction stops being enough.

## Portals: decoupling the DOM tree from the React tree

`createPortal(children, domNode)` renders `children` into a **different part of the DOM** — typically `document.body` — while keeping them in their original position in the *React* tree.

```jsx
import { createPortal } from 'react-dom';
function Modal({ children }) {
  return createPortal(children, document.body);
}
```

The defining property: **events bubble through the React tree, not the DOM tree.** A click inside the portal propagates to ancestors as defined by where `<Modal/>` sits in JSX, *not* by where the node physically lives in `document.body`. React's synthetic event system replays bubbling along the virtual tree. So a portal'd modal still triggers an `onClick` handler on the React-parent that rendered it, even though the DOM node is nowhere near that parent. Context also flows normally — the portal's children read the same providers as their JSX position dictates.

Why portals exist: CSS containment. A dropdown, tooltip, or modal rendered inside a parent with `overflow: hidden`, `transform`, or a low `z-index` gets clipped or mis-stacked. Portaling to `body` escapes the parent's stacking and clipping context while preserving the component's logical ownership, state, and event wiring. The mental model: **logical position (React tree) and physical position (DOM tree) are decoupled, and React keeps the logical one authoritative.**

A senior caveat: because events bubble through the React tree, a document-level "click outside to close" listener can fire on the same click that opened the portal, or treat clicks *inside* the portal as "outside" if you only check DOM ancestry. Check React-tree membership or stop propagation deliberately.

## Refs: a stable mutable container

A ref is an object `{ current: ... }` whose identity is stable across renders and whose mutation does **not** trigger a re-render. Two distinct uses, often conflated:

- **DOM refs.** Attach `ref` to a host element to get the underlying node after commit: `<input ref={inputRef}/>`, then `inputRef.current.focus()`. The ref is `null` during render and populated in the commit phase — never read `.current` during render expecting a node.
- **Instance variables.** `useRef` also stores any mutable value you want to persist across renders without causing re-renders — a timeout id, a previous value, a flag. It's the imperative cousin of state: state is *rendered*, a ref is *remembered*.

```jsx
const id = useRef(null);
useEffect(() => {
  id.current = setInterval(tick, 1000);
  return () => clearInterval(id.current);
}, []);
```

### Refs across components: React 19 changes the rules

Historically a function component couldn't receive a `ref` prop; you wrapped it in `forwardRef`. **React 19 makes `ref` an ordinary prop** for function components — `function Input({ ref }) {...}` works, and `forwardRef` is on a deprecation path. This removes a long-standing source of boilerplate.

For *controlled* imperative surfaces, `useImperativeHandle` lets a component expose a curated method set on its ref rather than the raw node — e.g. exposing `{ focus(), scrollToTop() }` while hiding the actual DOM. This is the disciplined way to offer an imperative API: a small, intentional contract instead of a leaked node.

## The escape-hatch philosophy: when to reach for the DOM

The declarative model covers *describable* UI: what the tree looks like for a given state. Some things are inherently **imperative and not expressible as state**:

- **Focus, text selection, and caret position** — there's no "is focused" you should drive purely from state for managed focus flows; you call `.focus()`.
- **Scroll position** — measuring and setting scroll is a DOM operation.
- **Media playback** — `video.play()` / `.pause()` are commands, not state you render.
- **Measuring layout** — reading `getBoundingClientRect()` for positioning a portal'd tooltip.
- **Integrating non-React libraries** — handing a node to D3, a map SDK, or a canvas library.

The rule of thumb: **use a ref when the operation is a verb (focus, scroll, play, measure) rather than a noun (a value the UI displays).** If you find yourself reading a value *out* of the DOM and treating it as the source of truth, you've crossed back into the imperative trap — derive from state instead. Refs are for talking *to* the DOM imperatively, not for storing application state in it.

## Senior pitfalls

- **Reading `ref.current` during render.** It's `null` until commit; layout reads belong in effects (`useEffect`/`useLayoutEffect`). Use `useLayoutEffect` specifically when you must measure-then-mutate before the browser paints to avoid a visible flicker.
- **Mutating a ref to force a render.** Refs don't trigger renders by design — if you need the UI to react, it's state, not a ref.
- **Treating portal DOM position as event scope.** Click-outside and focus-trap logic must account for React-tree bubbling, not DOM ancestry.
- **Overusing `useImperativeHandle`.** Most parent→child communication should be props/state; an imperative handle is a deliberate exception, not a convenience.

## See Also

- [[error-boundaries]]
- [[the-react-compiler]]
- [[server-vs-client-components]]
- [[declarative-ui]]
