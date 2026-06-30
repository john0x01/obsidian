# Rules Of Hooks

The two rules — call hooks only at the top level, and only from React functions or custom hooks — are not stylistic. They are a direct consequence of hooks having **order-based identity**: React maps a hook call to its persisted state purely by *which call number it is this render*. Break the order and you don't get a warning, you get one hook reading another hook's memory.

## Why The Rules Exist (Mechanically)

There is no key, no name, no `this.state.count`. During an update render React keeps a cursor and advances it one hook node per hook call (see [[hooks-internals]]). Identity = position in the linked list. So the *only* invariant React needs is: **the sequence of hook calls is identical on every render of a given component instance.**

Everything the rules forbid is a way to make that sequence vary:

- **Conditionals** — `if (x) useState()` adds/removes a node when `x` flips.
- **Loops with variable length** — changes node count.
- **Early returns before a hook** — truncates the list.
- **Calling hooks in event handlers / callbacks / `useMemo` bodies** — those run outside the render's dispatcher window; the dispatcher is the throwing no-op there, producing *"Invalid hook call."*

## The Two Failure Modes

```js
function Bad({ show }) {
  if (show) {
    const [a] = useState('a'); // hook0 only when show=true
  }
  const [b] = useState('b');   // hook0 when show=false, hook1 when show=true
}
```

- **Misalignment (silent corruption):** if the count *stays the same* but the meaning shifts, the cursor lands on a real node of the wrong hook. `b` reads `a`'s slot. No error, wrong value, near-impossible to debug from symptoms.
- **Count change (loud crash):** if the number of hooks differs between renders, React detects it: *"Rendered fewer hooks than expected. This may be caused by an accidental early return."* / *"Rendered more hooks…"*. This guard only fires on count *mismatch*, which is why misalignment-without-count-change is the scarier bug.

## The Linter Is Load-Bearing

`eslint-plugin-react-hooks` provides two rules:

- **`rules-of-hooks`** — enforces top-level calls and the naming heuristic. It identifies "is this a hook?" by the `use[A-Z]` naming convention and "is this a component/hook?" by capitalization / `use` prefix. This is *why naming matters*: a function named `usefetch` (lowercase f) won't be recognized as a hook; a component named `myComp` won't be recognized as a component. The lint is a static approximation of a dynamic invariant — it can have false negatives, so the rules still bind even where lint is silent.
- **`exhaustive-deps`** — a *different* concern (dependency arrays for effects/memo), often conflated. See [[effects]].

In React 19's tooling era the plugin gains awareness of the **React Compiler** (formerly "React Forget"); the compiler itself depends on the Rules of Hooks holding, and a component that violates them is bailed out of compilation rather than miscompiled.

## Custom Hook Composition

Custom hooks are the entire payoff of the rules. A custom hook is just a function that calls hooks; it has **no isolation** — its hook calls splice into the caller's list at the call site in order. This composes cleanly *because* the rules guarantee each custom hook contributes a fixed-shape, ordered subsequence:

```js
function useToggle(init) {
  const [on, setOn] = useState(init);          // appends 1 node
  const toggle = useCallback(() => setOn(o => !o), []); // appends 1 node
  return [on, toggle];
}
// Calling useToggle() twice appends two stable, ordered pairs. Safe.
```

The corollary: **you must call custom hooks unconditionally too**, because their internal hooks become part of your sequence. `if (x) useToggle()` is exactly as broken as `if (x) useState()`.

## Consequences Of Breaking Them

- **Heisenbugs:** values from one hook leak into another only on certain code paths; reproduction depends on prop/state combinations.
- **Compiler bailout:** the React Compiler refuses to optimize the offending component, silently losing its perf wins.
- **Future incompatibility:** any optimization that relies on stable hook identity (memoization, scheduling, dev-mode tooling) breaks.

## Senior Mental Model & Misconceptions

- The rules are *not* "React being opinionated." They are the minimum constraint that makes positional state work at all. The alternative design (keyed hooks) was considered and rejected as more verbose and error-prone.
- "I can break the rule if I'm careful and the count stays equal" — true mechanically, false in practice: the moment a teammate edits the branch, identity silently shifts. Treat the rules as inviolable.
- A conditional **value** is fine; a conditional **call** is not. `const x = cond ? a : b` after an unconditional `useState` is legal. Hoist the hook, branch the value.
- The one sanctioned conditional read is the new `use` API in React 19, which is *designed* to be callable conditionally — it is not subject to rule #1. See [[concurrent-hooks]].

## See Also

- [[hooks-internals]]
- [[effects]]
- [[concurrent-hooks]]
- [[refs]]
- [[01-programming-foundations/paradigms/closures|Closures]]
