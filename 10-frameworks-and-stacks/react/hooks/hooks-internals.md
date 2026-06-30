# Hooks Internals

Hooks are not magic and they are not stored "by name." A hook is a node in a singly-linked list hanging off the currently-rendering fiber, and your call order is the only thing that maps `useState` call #3 in your source to its persisted value. Understanding this list, the dispatcher swap, and `memoizedState` explains essentially every rule and every footgun in the entire hooks API.

## The Per-Fiber Linked List

Every component instance corresponds to a **fiber** (a unit of work, not a class instance). Each fiber has a `memoizedState` field. For function components, `fiber.memoizedState` does *not* hold "the state" — it holds a pointer to the **head of a linked list of hook objects**, one node per hook call in render order.

```
fiber.memoizedState ─▶ hook0 ─▶ hook1 ─▶ hook2 ─▶ null
                       (state)  (ref)    (effect)
```

Each hook node is roughly:

```js
const hook = {
  memoizedState,  // the hook's own state: state value, ref box, effect object, memo [value, deps]...
  baseState,      // base for update replay (state hooks)
  queue,          // update queue (state/reducer hooks)
  baseQueue,      // updates skipped/replayed across renders
  next,           // pointer to the next hook node
};
```

`memoizedState` is overloaded: for `useState`/`useReducer` it's the current value; for `useRef` it's `{ current }`; for `useMemo` it's `[value, deps]`; for `useEffect` it's an effect descriptor on a circular `updateQueue`. The hook *type* is implicit — there is no tag saying "this is a state hook." React relies entirely on **your call order being identical every render** to line node N up with the right consumer.

## The Dispatcher: Mount vs Update

`useState`, `useEffect`, etc. are thin wrappers that read `ReactCurrentDispatcher.current` and delegate. The dispatcher is a swappable object of hook implementations. React swaps it based on render phase:

- **Mount** (`HooksDispatcherOnMount`): each hook call *creates* a new node, appends it to the list, and runs first-render logic (e.g. compute the lazy initializer).
- **Update** (`HooksDispatcherOnUpdate`): each hook call *walks to the next existing node* via a cursor (`workInProgressHook`) and replays/derives the next value from its queue.

```
render() called
  dispatcher = isMount ? Mount : Update
  // every useX() reads dispatcher.useX
```

There is also a `HooksDispatcherOnMountWithHookTypes` / a "dev invalid" dispatcher set to `null` between renders. That null dispatcher is what produces the famous *"Invalid hook call. Hooks can only be called inside the body of a function component"* error: calling a hook outside render means `dispatcher` is the no-op that throws.

The cursor mechanism is why **order = identity**. On update, React doesn't search for "the useState that owns count." It advances the cursor one node per hook call. If you skip a hook (conditional) or add one, the cursor lands on the wrong node and your `count` value silently becomes your `name` state — or React throws *"Rendered fewer hooks than expected."* See [[rules-of-hooks]].

## How State Persists Across Renders

The fiber survives between renders (it lives on the fiber tree, double-buffered as `current` and `workInProgress`/`alternate`). Local variables in your function body are recreated every call and thrown away — they do **not** persist. What persists is the hook node's `memoizedState`, reachable because the *fiber* persists. So:

```js
function Counter() {
  const [count, setCount] = useState(0); // count = hook0.memoizedState this render
  // `count` is a render-local snapshot of a persisted value
}
```

`count` is a *binding to this render's value*, captured by closure. The next render produces a *new* `count` from the same hook node. This is the "state as a per-render snapshot" model that makes stale-closure bugs in effects make sense — detailed in [[state-and-reducers]] and [[effects]].

During an update, React clones the current fiber into `workInProgress` and the hook list is cloned lazily (`cloneUpdateQueue`), so updates accumulate on a queue rather than mutating in place. This double-buffering is what lets concurrent React **throw away** a render that gets interrupted: the `current` tree is untouched until commit.

## Mental Model & Senior Pitfalls

- Think of hooks as **array slots indexed by call order**, not as named storage. The classic teaching trick (`useState` as `arr[i++]`) is essentially accurate.
- "Why can't I call hooks in a loop with variable iterations?" Because node count must be stable across renders; a variable-length loop changes the list shape. A *constant*-length loop is technically fine but fragile and lint-flagged.
- Custom hooks don't get their own list. They **splice their hook calls into the caller's list** at the call site. Two `useFetch()` calls in one component append two independent runs of `useFetch`'s internal hooks, in order. There is no isolation boundary.
- The dispatcher swap is also how **React Server Components / RSC and `react-dom/server`** ship different hook implementations (e.g. effects are no-ops on the server) — same call sites, different dispatcher.
- `memoizedState` being untyped is *deliberate*: it keeps the per-hook overhead to one object and one pointer, which matters because a large app allocates millions of these.

## See Also

- [[rules-of-hooks]]
- [[state-and-reducers]]
- [[effects]]
- [[refs]]
- [[02-data-structures-and-algorithms/linked-lists/linked-lists|Linked Lists]]
- [[01-programming-foundations/paradigms/closures|Closures]]
- [[fiber-architecture]] — hooks live on the fiber
