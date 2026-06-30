# State And Reducers

`useState` is not a variable — it is a subscription to a per-hook **update queue** that React drains during render to produce this render's snapshot. `useReducer` is the same machinery with the reducer made explicit; `useState` is literally implemented as `useReducer` with a fixed "basic state reducer." Internalizing the queue model explains batching, functional updates, bailouts, and why state "doesn't update immediately."

## State As A Per-Render Snapshot

A render is a pure function of props + the state values React hands you *for that render*. `count` is a constant binding captured by closure; calling `setCount` does not mutate it. It schedules an update and triggers a re-render, which runs your function again and produces a *new* `count`.

```js
const [count, setCount] = useState(0);
setCount(count + 1);
setCount(count + 1);
console.log(count); // still 0 — this render's snapshot is frozen
// Next render: count = 1, NOT 2 — both updates computed from the same stale 0
```

This is the root of most "state is one step behind" confusion. The fix is functional updates (below), which compute from the *queued* value rather than the captured snapshot.

## The Update Queue

Each state/reducer hook node owns a `queue` with a circular linked list of pending **update** objects. An update is roughly `{ action, eagerState, lane, next }`. `setState(action)` (where `action` is a value *or* an updater function) enqueues an update and schedules work on the fiber.

On the next render, the reducer replays the queue over `baseState`:

```
newState = baseState
for each update in queue (in lane priority):
   newState = reducer(newState, update.action)
hook.memoizedState = newState
```

For `useState`, the "reducer" is the **basicStateReducer**: `(state, action) => typeof action === 'function' ? action(state) : action`. That single line *is* the difference between value updates and functional updates — a functional update is just an action that happens to be a function applied to the running accumulator.

This is why **`useState` is sugar over `useReducer`.** Same node shape, same queue, same replay; only the reducer differs.

## Functional Updates

```js
setCount(c => c + 1);
setCount(c => c + 1); // now +2: each updater receives the accumulated value
```

Because the reducer threads `newState` through each queued action, function-updaters compose correctly even within one batch and across stale closures. Rule of thumb: **if the next state depends on the previous state, use the functional form** — it is closure-immune (the updater doesn't capture `count`).

## Batching

React groups multiple `setState` calls into one render. Pre-18 this only happened inside React event handlers. **React 18+ uses automatic batching everywhere** — promises, `setTimeout`, native event handlers, microtasks. All `setState` calls in the same tick enqueue updates and schedule *one* render. `flushSync(() => setState(...))` opts a specific update out, forcing a synchronous render+commit (rarely needed; mainly for measuring DOM between updates, pairs with [[refs]] and `useLayoutEffect` in [[effects]]).

## Bailout via Object.is

Two bailout layers:

1. **Eager bailout (pre-render):** if a state hook's queue is empty *before* you dispatch, React computes the new state eagerly and compares with `Object.is`. Equal → it skips scheduling a re-render entirely. This is why `setCount(0)` when count is already `0` may not even re-render.
2. **Render bailout:** if, after rendering, a fiber's state is referentially equal and props haven't changed, React can skip re-rendering children (the "bailout" path on the fiber).

`Object.is` (not `===`) is the comparator — it treats `NaN` as equal to itself and distinguishes `+0`/`-0`. The practical consequence: **mutating an object and setting it back gives the same reference → bailout → no update.** Always produce new references. See [[01-programming-foundations/paradigms/immutability|Immutability]].

```js
state.items.push(x); setItems(state.items); // same ref → Object.is true → bailout, no render
setItems(prev => [...prev, x]);             // new ref → renders
```

## Lazy Initialization

```js
const [s] = useState(() => expensiveInit()); // initializer runs ONCE, on mount only
const [s] = useState(expensiveInit());        // runs EVERY render, result discarded after mount
```

Pass a function to defer expensive initial computation to the mount render only. Same for `useReducer`'s third argument: `useReducer(reducer, initialArg, init)` runs `init(initialArg)` once. The inline-call form is a common senior-level perf leak hiding in plain sight.

## useState vs useReducer — When

- **useReducer** when next state depends on complex prior state, when multiple values update together, when update logic is worth testing in isolation, or when you want to pass `dispatch` down (it's stable, dodging callback churn — see [[memoization]]).
- **useState** for independent, simple values. It is genuinely the same engine, so "switch for performance" is a myth; switch for *readability and update locality*.

## Senior Pitfalls

- Reducers must be **pure** — React may call them twice in StrictMode/dev to surface side effects (see [[effects]]).
- Don't store derived data in state; compute during render. State should be the minimal source of truth.
- The eager-bailout `Object.is` check still re-runs your component once after the *first* `setState` even on equal values in some paths; don't rely on bailout for correctness, only as an optimization.

## See Also

- [[hooks-internals]]
- [[effects]]
- [[memoization]]
- [[concurrent-hooks]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
- [[01-programming-foundations/paradigms/pure-functions|Pure Functions]]
