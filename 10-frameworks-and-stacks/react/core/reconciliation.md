# Reconciliation

Reconciliation is the algorithm that takes a freshly rendered element tree and the existing fiber tree and computes *what changed*, so React can apply the minimal set of host mutations. A general tree-diff is O(n³); React makes it **O(n)** by adopting two heuristics that hold for real UIs, and by leaning on developer-supplied `key`s for the one case the heuristics can't solve cheaply: reordering siblings.

## The two heuristics (and the O(n) assumption)

The optimal edit distance between two arbitrary trees of n nodes is O(n³) — unusable. React replaces optimality with two assumptions that are almost always true for component UIs:

1. **Different element `type` ⇒ different tree.** If the new element at a slot has a different `type` than the existing fiber (e.g. `<div>` became `<span>`, or `<Profile>` became `<Settings>`), React does **not** try to reconcile their children. It tears down the entire old subtree (firing unmounts/cleanups, destroying state and DOM) and mounts the new one from scratch. The bet: people rarely transform a `<div>` subtree into a `<span>` subtree while wanting to preserve internals.
2. **Stable children are matched by `key`; otherwise by position.** Within a set of siblings, React pairs new elements to old fibers either by `key` (if provided) or by index order. This makes list diffing linear instead of computing a true minimal edit script.

Together these turn the diff into a single pass that compares like-for-like at each position. The O(n) is *because* React refuses to consider moves across the tree — it only ever compares a slot to the same slot.

## Same-type update vs different-type replace

At each matched slot:

```
new.type === fiber.type ?
   → UPDATE: keep the fiber, keep its state/hooks, update props,
             recurse into children (compute new props for host node)
   → REPLACE: unmount old fiber subtree (cleanup effects, lose state),
              mount new fiber subtree
```

This is the rule behind two of React's most consequential behaviors:

- **State preservation across renders** works *because* the fiber survives when `type` matches at the same position. Your `useState` lives on the fiber; an update keeps it.
- **Unexpected state loss / remounts** happen when `type` *or position* changes. Switching between two different component types at a slot, or conditionally rendering `cond ? <A/> : <B/>` where A and B differ in type, remounts. So does defining a component inside another (new function identity each render → new `type` → remount).

## Keys and list diffing

For lists, React must answer "which new item is which old item?" Without keys it falls back to **index matching**: new[0]↔old[0], new[1]↔old[1]. That's fine for append-only lists but catastrophic for insert/reorder/delete:

```
items: [A, B, C]   keys: index        prepend X → [X, A, B, C]
old:   0:A 1:B 2:C
new:   0:X 1:A 2:B 3:C
index match: 0:A→X (mutate), 1:B→A (mutate), 2:C→B (mutate), 3:add C
  ⇒ React mutates every row + remounts subtree state at each slot
```

With **stable, identity-based keys** (`key={item.id}`), React matches new `X` as a *new* fiber and reuses A/B/C's existing fibers by key, performing one insert and zero content mutations. Keys let React track *element identity across reorders* so that component state, DOM focus, and effects follow the right logical item.

Rules that fall out of this:

- Keys must be **stable** (same logical item ⇒ same key across renders), **unique among siblings**, and **not derived from index** when the list can reorder/insert/delete.
- `Math.random()` or array index as key are the two classic bugs — random keys remount everything every render; index keys silently attach the wrong state/DOM to the wrong row after a reorder.
- Keys are scoped to their sibling set, are stripped out of `props` by the compiler (they're reconciliation metadata, not data), and never reach the component.

## Bailout conditions

React tries hard to *not* re-render and *not* recurse:

- **Same element reference.** If a child element is referentially identical to last render (e.g. an element passed through `props.children`, or hoisted), React bails out of re-rendering that subtree — the description provably didn't change. This is why composition-via-children is a perf pattern.
- **`React.memo` / props equality.** A memoized component bails if its props are shallow-equal to the previous props. The (React 19) **React Compiler** auto-inserts much of this memoization, so manual `memo`/`useMemo` becomes less necessary.
- **State unchanged.** If a `useState`/`useReducer` setter is called with a value `Object.is`-equal to the current state, React may bail out of re-rendering that component (it may still render it once to confirm, then bail on the children).
- **Unchanged context.** A subtree not reading a changed context, with equal props, bails.

These bailouts are *short-circuits on the same heuristics* — they let whole subtrees be skipped while preserving the same-slot/same-type matching for the parts that do change.

## Senior pitfalls and misconceptions

- **"Keys improve performance."** Their primary job is *correctness of identity*; the performance win is a consequence. The bug index-keys cause is wrong state on the wrong row, which looks like data corruption, not slowness.
- **"Same type guarantees no work."** It guarantees the fiber survives, but children still reconcile; a same-type parent with changed children still diffs and may commit.
- **Resetting state via key.** You can *intentionally* force a remount by changing `key` on a component — the canonical way to reset a subtree's state when an identity changes (e.g. `key={userId}`).
- **Reconciliation ≠ commit.** Diffing computes the patch; committing applies it. A clean diff produces no DOM ops even though reconciliation ran.

## See Also

- [[virtual-dom]]
- [[elements-vs-components]]
- [[declarative-ui]]
- [[jsx]]
- [[10-frameworks-and-stacks/react-native/performance/re-renders|RN Re-renders]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
