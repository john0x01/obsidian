# Transitions

A **transition** marks a state update as *non-urgent*, telling React it may be interrupted, deferred, and rendered in the background while the UI stays responsive to more pressing work. `useTransition` and `startTransition` are the API; underneath, they assign the update a low-priority **transition lane** (see [[lanes-and-priorities]]), which the concurrent renderer is then free to preempt. The core idea: separate "the state changed instantly" (urgent — typing, clicking) from "the consequences of that change" (non-urgent — re-rendering an expensive view), so the former never waits on the latter.

## The API

```jsx
const [isPending, startTransition] = useTransition();

function onChange(e) {
  setQuery(e.target.value);              // urgent: keep the input responsive
  startTransition(() => {
    setResults(filter(allItems, e.target.value)); // non-urgent
  });
}
```

- **`startTransition(fn)`** — the state updates triggered *synchronously* inside `fn` are tagged as transitions. (The standalone `startTransition` import is the same, without `isPending`.)
- **`isPending`** — `true` while the transition's background render is in flight, so you can show a subtle pending indicator without blocking.

In React 19, `startTransition` also supports **async functions** (an "action"): awaited work inside it stays within the transition, and `useActionState`/form actions build on this to manage pending/error state for async submissions.

## Mechanism: Lanes, Interruption, Entanglement

When you call `startTransition`, React sets an internal flag so any `setState` during `fn` requests a **transition lane** rather than the default lane. Concretely:

1. The urgent update (e.g. `setQuery`) lands on a high-priority lane and renders first — the input updates immediately.
2. The transition update lands on a transition lane and is scheduled at lower priority.
3. While React renders the transition's expensive tree, the work loop yields to the browser between fibers; if another urgent update arrives (another keystroke), it **interrupts** the in-progress transition render, which is discarded and later restarted with fresh state.
4. The transition's updates are **entangled** so they render as one consistent unit — you never commit a half-updated view.

This is why typing into a filtered list feels smooth: each keystroke updates the input urgently and *throws away* the now-stale background render of the list, only committing once you pause.

## `useDeferredValue`

`useDeferredValue` is the value-level counterpart to the action-level `startTransition`. You pass it a value; it returns a version that "lags behind" during urgent updates:

```jsx
const deferredQuery = useDeferredValue(query);
// pass deferredQuery to the expensive child
```

On an urgent update, React first re-renders with the *old* deferred value (fast), then schedules a background re-render with the new value at transition priority. The expensive child re-renders only in the background pass. Choose `useDeferredValue` when you don't control the `setState` call site (e.g. the value comes from props or a parent); choose `startTransition` when you own the update and can wrap it. React 19 also lets you pass an initial value: `useDeferredValue(value, initialValue)`.

## UX Patterns

- **Search/filter over large lists** — urgent input, deferred results; the canonical case.
- **Tab/route switching** — wrap the navigation in a transition so the current screen stays visible (and interactive) until the new one is ready, coordinating with Suspense to avoid fallback flicker (see [[suspense]]).
- **Pending affordances** — use `isPending` to dim/disable or show a spinner *over* the stale content, signaling "working" without yanking the UI away.
- **Optimistic UI** — `useOptimistic` layers an immediate optimistic state on top of a pending async transition/action.

```text
keystroke ─▶ [urgent] setQuery        ─▶ input repaints now
          └▶ [transition] setResults  ─▶ renders in bg, preemptible
                                           isPending = true …→ commit
```

## Senior Pitfalls

- **Only synchronous updates inside `startTransition` are marked.** A `setState` inside a `setTimeout` or a `.then()` callback created within `fn` is *not* a transition (the flag is gone by the time it runs) — unless you use the async-action form correctly. Tag the actual update, not just the scheduling.
- **Transitions don't make slow code fast.** They prevent the expensive render from *blocking input*; the work is still done. Pair with memoization/virtualization, or the background render itself stays heavy and `isPending` lingers.
- **`useDeferredValue` shows stale data on purpose.** Without a visual staleness cue, users perceive unexplained lag. Also: deferring a value that feeds a cheap component is pointless overhead.
- **Don't wrap urgent updates.** Putting the input's own `setState` in a transition makes typing feel laggy — the input itself becomes interruptible. Keep the directly-controlled value urgent; defer only the downstream cost.
- **Transitions and Suspense are a team.** A transition update that suspends keeps the old UI instead of showing the fallback — relying on this is the intended pattern for navigations.

## See Also

- [[lanes-and-priorities]]
- [[concurrent-rendering]]
- [[suspense]]
- [[work-loop-and-scheduling]]
- [[10-frameworks-and-stacks/react-native/performance/re-renders|RN Re-renders]]
