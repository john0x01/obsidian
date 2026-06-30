# State Architecture

State architecture is the discipline of deciding **where each piece of state lives, who owns it, and whether it should exist at all**. Most React performance and correctness pain is not a rendering problem — it is a state-placement problem. The senior skill is treating state as a liability to be minimized, not a feature to be accumulated, and routing each value to the layer whose lifecycle and access pattern matches it.

## Stored vs derived state

The first cut is whether a value is *primitive* (cannot be computed from anything else) or *derived* (a pure function of other state). Derived values must **not** be stored.

```jsx
// ❌ two sources of truth that can drift
const [items, setItems] = useState([]);
const [count, setCount] = useState(0); // duplicate of items.length

// ✅ count is derived, computed at render
const [items, setItems] = useState([]);
const count = items.length;
```

Storing a derivation creates a synchronization obligation: every writer of `items` must remember to update `count`, and the moment one forgets, you have a bug that no type checker catches. The rule: **if you can compute it during render, do.** Reach for `useMemo` only when the computation is provably expensive *and* shows up in a profile — memo is a cache, and a cache is itself stored state with its own invalidation hazard. A common senior mistake is memoizing cheap derivations "to be safe," paying allocation and comparison cost for nothing.

## The single source of truth

Every fact should have exactly one authoritative home. Duplication is the root cause of "the two panels disagree" bugs. This applies across categories too: if the URL says `?tab=billing` and a `useState('billing')` also exists, you now have two truths and a sync bug waiting on back-button navigation. Pick one owner and let everything else read from it.

A subtle corollary: **the DOM is not state.** Reading `inputRef.current.value` and treating it as canonical reintroduces an imperative source of truth that React doesn't manage. Either let React own the value (controlled) or accept the DOM as owner (uncontrolled) — never both for the same field.

## Colocation, then lifting

Default placement is **as local as possible** — colocated in the component that uses it. Local state is the cheapest, most encapsulated kind: it has the lifecycle of the component, can't be mutated from afar, and re-renders only its subtree.

You lift state up **only when two siblings must agree**, and you lift it to their *lowest common ancestor* — no higher. Lifting too high is the dominant cause of accidental re-render fan-out: a value used by two leaves gets parked at the route root, and now every keystroke re-renders the whole page. The mirror mistake is lifting too low and then prop-drilling a setter five levels down. The corrective tools, in escalating order:

- **Composition / children-as-props** — pass JSX through, so an expensive subtree renders in the parent's scope and isn't re-rendered by the stateful middle.
- **Context** — for genuinely tree-wide values (theme, auth, locale) that change rarely. Context is a *broadcast*, not a store: every consumer re-renders on any value change, so splitting contexts by update frequency matters.
- **External store** — when many distant components read/write fast-changing shared state and Context's all-or-nothing re-render becomes the bottleneck.

## The state categories

Treat "state" as several distinct problems with different correct homes:

- **Server / cache state** — data owned by a backend (lists, entities). It is *remote, shared, and stale by default*. This is not really client state; it needs caching, revalidation, dedup, and background refetch. Hand it to a query library (or RSC + `use`), never `useEffect`+`useState`, which reinvents caching badly and causes waterfalls and races.
- **URL state** — anything that should survive reload, be shareable, or participate in browser history: current tab, filters, pagination, selected id. The URL is a free, persistent, server-readable store. Underusing it is a classic mistake; over-storing ephemeral UI bits in it is the opposite one.
- **Form state** — transient, high-frequency, locally owned input values plus validation/dirty/submission status. Often best kept uncontrolled or in a dedicated form library to avoid re-rendering the tree on every keystroke.
- **Client UI state** — open/closed, hovered, selected-in-memory. Genuinely local, ephemeral, `useState`/`useReducer` territory.

Mixing categories is where architectures rot: caching server data in Redux and manually invalidating it, or driving a modal's open state through the network layer.

## Senior pitfalls

- **Effect-syncing state from props.** `useEffect(() => setX(prop), [prop])` is a derived-state smell — compute during render, or key the component to reset it. React 19's docs codify this under "you might not need an effect."
- **Reducer vs useState.** Reach for `useReducer` when next state depends on previous state through several actions, or when update logic is worth testing in isolation — not as a default.
- **Context as a store.** Context distributes a value; it has no selector mechanism, so it can't prevent re-renders on partial change. That gap is precisely why external stores exist.

The philosophy is subtractive: the best state is no state (derive it), the next best is local state (colocate it), and shared mutable state is the thing you push outward and minimize, because every shared writable value is a synchronization contract you now have to honor forever.

## See Also

- [[external-stores]]
- [[rsc-model]]
- [[actions-and-forms]]
- [[declarative-ui]]
