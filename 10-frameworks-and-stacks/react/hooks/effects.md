# Effects

The single most important reframing: `useEffect` is **not** a lifecycle hook. It is a tool for **synchronizing** a piece of the outside world (a subscription, the DOM, a network resource, a timer) with your render output. You don't think "run this on mount/update/unmount"; you think "this external thing should match this state — keep them in sync, and undo when the inputs change." Everything about deps, cleanup, and timing falls out of the synchronization model.

## Synchronization, Not Lifecycle

An effect describes a *relationship*: "given these dependency values, the external system should be in this state." React's job is to make reality match after every commit where the relationship's inputs changed. The lifecycle framing ("componentDidMount") leads you astray because it makes you ask "when does this run?" The right question is "what is this synchronizing, and what does the dependency array say its inputs are?"

```js
useEffect(() => {
  const conn = connect(roomId);     // set up: sync to roomId
  return () => conn.disconnect();   // cleanup: tear down the OLD sync
}, [roomId]);                       // input changed → tear down old, set up new
```

When `roomId` changes, React doesn't "update" the connection — it runs cleanup for the *previous* room, then re-runs setup for the new one. Cleanup is "undo the last synchronization," not "componentWillUnmount."

## The Dependency Array & Stale Closures

The effect function is a **closure created fresh every render**, capturing that render's props/state. The dependency array is your *claim* about which captured values the effect reads. React compares deps with `Object.is`; if all equal, it **skips** running the effect and keeps the previous closure alive.

The stale-closure bug: omit a dep, and the effect keeps running an old closure that captured an old value.

```js
useEffect(() => {
  const id = setInterval(() => setCount(count + 1), 1000); // captures count=0 forever
  return () => clearInterval(id);
}, []); // lying: effect reads `count` but it's not declared
// Fix: setCount(c => c + 1) — functional update removes the dependency
```

`exhaustive-deps` (from `eslint-plugin-react-hooks`) enforces honesty about deps. **Do not suppress it to "fix" a re-run loop** — that's treating a symptom. The real fixes: functional updates ([[state-and-reducers]]), moving the function inside the effect, `useCallback`/`useMemo` on dependencies ([[memoization]]), or refs for values you want to read-but-not-react-to ([[refs]]). An empty `[]` is a *claim that the effect depends on nothing reactive* — often a lie.

## Cleanup Timing

```
mount:        setup
update (deps changed):  cleanup(prev) → setup(next)
unmount:      cleanup
```

Cleanup runs **before** the next setup and on unmount. Critically, the cleanup closure captures the *render in which it was created*, so it tears down exactly what that render set up — no stale-handle leaks if deps are honest. Missing/incorrect cleanup is the #1 source of leaked listeners, duplicate subscriptions, and racey fetches (use an `ignore` flag or `AbortController`).

## useEffect vs useLayoutEffect vs useInsertionEffect

All three share the dependency/cleanup model; they differ in **commit-phase timing**:

- **`useInsertionEffect`** — fires *before* DOM mutations are read by layout effects; meant for CSS-in-JS libraries to inject `<style>` rules before layout reads. You almost never use it directly. Cannot read refs/layout.
- **`useLayoutEffect`** — fires **synchronously after DOM mutations but before the browser paints**. Use when you must measure DOM and mutate before the user sees a flicker (e.g. position a tooltip from a measured rect). It blocks paint, so keep it cheap. Pairs with `flushSync`.
- **`useEffect`** — fires **asynchronously after paint** (passive effect). The default. Doesn't block visual updates.

```
commit → mutate DOM → useInsertionEffect → useLayoutEffect → browser paint → useEffect
```

Choosing `useLayoutEffect` to "fix a flicker" is correct; reaching for it by default is a perf mistake — you serialize work that could happen after paint.

## Effect Ordering

Within a single commit, effects fire in **tree order** (parent's effect cleanup/setup interleaved with children per React's traversal — children's passive effects generally run before parents' in the commit's passive phase). Across a tree, all `useLayoutEffect`s of the committed tree run (synchronously) before the browser paints; all `useEffect`s run later in a batched passive phase. Don't rely on cross-component effect ordering for correctness — model dependencies via props/state instead.

## StrictMode Double-Invocation

In dev, `StrictMode` **mounts, unmounts, then remounts** every component once: `setup → cleanup → setup`. This is intentional — it surfaces effects that aren't resilient to being run twice (missing cleanup, non-idempotent setup, subscriptions that leak). It does **not** run in production. If your effect breaks under double-invoke, your cleanup is wrong, not StrictMode. This is the strongest practical pressure toward writing effects as idempotent synchronizations.

## The Correct Mental Model & Pitfalls

- Ask "what external system am I synchronizing?" If the answer is "nothing — I'm transforming data," **you probably don't need an effect.** Compute during render; derive, don't sync. Effects for derived state are a classic anti-pattern (extra render, stale windows).
- Don't use effects to *react to events* — handle them in the event handler. Effects are for synchronizing with things that exist *because* the component is rendered, independent of any specific user action.
- Fetching in effects is acceptable but fragile (races, no caching, waterfalls); prefer a data library or RSC/`use` ([[concurrent-hooks]]). If you do, always guard against out-of-order resolution.
- An effect with `[]` is not "componentDidMount" — under StrictMode and future features it can run more than once. Idempotency is mandatory.

## See Also

- [[state-and-reducers]]
- [[refs]]
- [[memoization]]
- [[concurrent-hooks]]
- [[01-programming-foundations/paradigms/closures|Closures]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
