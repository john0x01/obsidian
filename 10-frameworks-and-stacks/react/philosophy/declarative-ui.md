# Declarative UI

React's foundational bet is that a UI is best expressed as a *pure function of state* — `UI = f(state)` — rather than as a sequence of mutations applied to a live tree. You describe **what** the screen should look like for a given state; React figures out **how** to make the real DOM match. This single inversion is what unlocks reconciliation, time-travel debugging, and concurrent rendering, all of which would be intractable against hand-written imperative DOM code.

## Imperative vs declarative as a mental model

In an imperative model you hold a reference to a node and issue commands against it over time: `el.textContent = count; if (count > 5) el.classList.add('hot')`. The correctness of the *next* frame depends on the *exact history* of mutations that produced the current frame. State lives implicitly, smeared across the DOM and across closures. Every event handler must know the current DOM shape to patch it correctly.

Declarative rendering collapses that history. You write a function that, given the current state, returns the *entire intended* tree:

```jsx
function Counter({ count }) {
  return <span className={count > 5 ? 'hot' : ''}>{count}</span>;
}
```

There is no "add class" — only "the class *is* this for this state." The component never reasons about transitions; it only describes endpoints. React owns the transition (the diff). This is the same shift as SQL over manual cursor loops, or HTML over `document.write` calls: you surrender control of the *procedure* to gain control of the *specification*.

## Why idempotent render is the load-bearing constraint

For `f(state)` to be meaningful, render must be **idempotent and side-effect-free**: calling it twice with the same props/state yields the same element tree and mutates nothing observable. React leans on this hard.

- It lets React call your component **as often as it wants** — to compute a diff, to discard work, to retry. React 19's StrictMode double-invokes renders in dev precisely to surface code that violates this.
- It lets React **throw away** a render entirely (an interrupted concurrent render) with zero cleanup, because nothing happened except producing a value.
- It makes render **order-independent and resumable**: render is the "pure" phase; all the impure work (DOM mutation, effects, refs) is deferred to a separate commit phase that React schedules.

This is why mutating props, reading `Date.now()` for layout, or writing to a ref *during* render are bugs in waiting — they break the contract that render is a pure projection of state.

## What declarativeness buys downstream

**Reconciliation.** Because each render produces a full description, React can compare the new tree to the previous one and derive the minimal mutation set. The diff is a *derived artifact*, not something you author. You couldn't do this against imperative code — there's no "previous intended tree" to diff against, only mutation side effects.

**Concurrency.** Render being pure and interruptible is the precondition for concurrent rendering (`useTransition`, Suspense). React can render a tree partway, pause to handle a higher-priority update, and resume or restart — only safe because rendering produced no externally visible effects.

**Time-travel & predictability.** Since the view is a deterministic function of state, replaying a sequence of states reproduces the exact UI. Devtools, hot reload that preserves state, and undo stacks all exploit this. The bug surface shrinks: "the UI is wrong" reduces to "the state is wrong," not "some mutation fired in the wrong order."

## Senior pitfalls and misconceptions

- **"Declarative means slow / re-renders everything."** No. Render produces a *cheap in-memory description*; the expensive DOM work is gated by the diff. The cost model is "describe everything, mutate the minimum." Misunderstanding this leads to premature `memo` everywhere.
- **The view is declarative; effects are not.** `useEffect` is the imperative escape hatch — an acknowledgment that some things (subscriptions, focus, network) are inherently procedural. The skill is keeping the *describable* part declarative and quarantining imperative work into effects/refs.
- **Declarative ≠ no control flow.** You still use `if`/`map`. The distinction is that control flow computes *which description to return*, not *which mutation to perform*.
- **State is the single source of truth, not the DOM.** Reading layout state back out of the DOM and treating it as canonical reintroduces the imperative trap. Derive, don't read back.

The deeper philosophy: React treats the DOM as an *output device*, not a data structure you collaborate on. You own state; React owns the mapping from state to pixels. Everything else in React's architecture is downstream of refusing to let you touch the DOM directly.

## See Also

- [[unidirectional-data-flow]]
- [[reconciliation]]
- [[virtual-dom]]
- [[01-programming-foundations/paradigms/declarative-programming|Declarative Programming]]
- [[01-programming-foundations/paradigms/imperative-programming|Imperative Programming]]
- [[01-programming-foundations/paradigms/pure-functions|Pure Functions]]
