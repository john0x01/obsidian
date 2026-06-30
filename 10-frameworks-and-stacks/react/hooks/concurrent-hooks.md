# Concurrent Hooks

React 18+ made rendering **interruptible**: a render can be started, paused, abandoned, or replayed at a different priority before it commits. The concurrent hooks — `useTransition`, `useDeferredValue`, `useSyncExternalStore`, `useId`, and React 19's `use` — are the public surface for cooperating with that scheduler. They exist to keep the UI responsive while expensive renders happen, and to keep external data consistent when renders can be torn apart and redone.

## The Substrate: Lanes & Interruptible Rendering

React assigns each update a **lane** (priority). High-priority updates (typing, clicks) can preempt low-priority ones (filtering a huge list). Because a low-priority render may be interrupted and **its in-progress fiber tree discarded** before commit (the `current` tree stays untouched — see [[hooks-internals]]), render must be pure and replay-safe. This is also why effects, not render, are the place for side effects ([[effects]]) and why external reads need special handling.

## useTransition

Marks an update as **non-urgent**, letting urgent updates interrupt it:

```js
const [isPending, startTransition] = useTransition();
startTransition(() => setQuery(input)); // this render is interruptible / low-priority
```

`isPending` is true while the transition render is in flight, so you can show a subtle pending state *without* blocking. The classic use: keep the input responsive while an expensive results list re-renders at lower priority. The update inside `startTransition` won't block typing; React renders the heavy tree in the background and commits when ready (or throws it away if a newer urgent update arrives). The callback must be synchronous for the state update to be marked. **React 19** also brings **Actions**: `startTransition` can wrap async functions, and `useActionState`/`useFormStatus`/`useOptimistic` build on transitions to manage pending/error/optimistic states for async mutations and forms.

## useDeferredValue

The "pull" counterpart to `useTransition`'s "push." Instead of marking the *update* as non-urgent, you mark a *value* as allowed to lag:

```js
const deferred = useDeferredValue(value); // deferred trails `value` under load
const list = useExpensiveList(deferred);
```

React renders once with the old value (fast, urgent) and schedules a second render with the new value at lower priority; if a newer value arrives first, the stale render is dropped. Use it when you *consume* a value you don't control (e.g. a prop) and can't wrap its setter in a transition. React 19 adds an optional initial value: `useDeferredValue(value, initialValue)` for the first render. Pair it with `React.memo` on the expensive child so the deferred render actually skips work when the value hasn't changed yet.

`useTransition` vs `useDeferredValue`: transition when you **own the state update**; deferred value when you only **receive the value**.

## useSyncExternalStore & Tearing

Concurrent rendering can read an external mutable store at different times during a single interruptible render, so two components could render with **different versions of the same store** — *tearing*: visible inconsistency in one frame. `useSyncExternalStore` is the contract that prevents it:

```js
const snapshot = useSyncExternalStore(subscribe, getSnapshot, getServerSnapshot?);
```

- `subscribe(cb)` — register `cb` to fire on store change; returns an unsubscribe.
- `getSnapshot()` — return the current value; **must be cached / referentially stable** when unchanged (returning a fresh object each call causes infinite re-renders).
- `getServerSnapshot()` — value for SSR/hydration.

React forces a **synchronous, consistent re-read** when the store changes, opting external state out of concurrent time-slicing so every component in a commit sees the same snapshot. This is *the* official integration point for any external store (Redux, Zustand, etc. use it internally) — do not subscribe to external mutable sources via ad-hoc effects in concurrent mode; you risk tearing. Contrast with [[context]], which is tear-free because it flows through React's own render.

## useId

Generates a **stable, unique, hydration-safe id** consistent between server and client:

```js
const id = useId(); // e.g. ":r1:" — same on server and client for this position
<label htmlFor={id}>…</label><input id={id} />
```

It encodes the component's position in the tree so SSR and client agree, avoiding hydration mismatches. It is **not** for list keys (it's per-component-instance, not per-list-item) and not random — its whole value is determinism across the server/client boundary.

## use (React 19)

`use` reads a resource — a **Promise** or a **Context** — and is the first hook deliberately **exempt from the top-level rule**: it *may* be called conditionally and inside loops/branches ([[rules-of-hooks]]).

```js
const value = use(somePromise);   // suspends until resolved, integrates with Suspense
const theme = use(ThemeContext);  // like useContext, but conditional-safe
if (cond) { const x = use(ctx); } // legal
```

- Reading a **Promise** with `use` suspends the component (showing the nearest Suspense fallback) until it resolves, then resumes with the value. It's the idiomatic way to consume async data from Server Components / a data layer, replacing much effect-based fetching. The promise should be created outside render or cached (a promise created inline each render is a new one each time — typically passed down from a Server Component or a cache).
- Reading **Context** with `use` behaves like `useContext` but, being conditional-capable, lets you early-read or branch.

`use` isn't a "use*" hook in the traditional sense — it's a first-class primitive the renderer understands, which is how it can legally break rule #1: React unwinds and replays the component when the resource isn't ready, so a conditional `use` doesn't corrupt the positional hook list.

## Senior Mental Model

- Concurrency is about **priority and interruption**, not parallelism (it's still single-threaded — [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]). These hooks let you tell React what's urgent and what can wait or lag.
- Tearing is a *concurrency* bug class; `useSyncExternalStore` is its dedicated fix. If you read mutable external state any other way, assume you can tear.
- Don't sprinkle transitions everywhere — mark the genuinely heavy, non-urgent updates. Overuse adds scheduling overhead with no UX win.

## See Also

- [[hooks-internals]]
- [[effects]]
- [[context]]
- [[rules-of-hooks]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
- [[01-programming-foundations/paradigms/reactive-programming|Reactive Programming]]
- [[concurrent-rendering]] — what these hooks drive
