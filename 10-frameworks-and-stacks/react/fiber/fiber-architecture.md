# Fiber Architecture

A **fiber** is a plain JavaScript object that represents a single unit of work in React's reconciler. Collectively the fibers form a tree that mirrors your component tree, but the real reason fibers exist is subtler: they are a re-implementation of the **call stack in user space**. The native call stack is fast but cannot be paused, inspected, or resumed; React needed exactly those abilities to render incrementally, so it modeled the stack as a heap-allocated, mutable data structure it fully controls.

## Why Reify The Stack

In the legacy (pre-16) "stack reconciler," reconciling a tree was a recursive descent: render a component, recurse into its children, unwind. Because it rode the native stack, once started it ran to completion — no yielding, no prioritization, no abandoning half-done work. A large update blocked the main thread and dropped frames. Fiber breaks that recursion into a loop over discrete nodes. Each fiber holds the state a stack frame would (which component, what props, where to return) so React can stop after any node, hand the thread back to the browser, and later resume exactly where it left off. That is what "interruptible, resumable" work means concretely.

## The Fiber Node

A fiber instance carries roughly these fields (names from the React source):

- **`type`** and **`elementType`** — the function/class/host-tag this fiber renders (`App`, `'div'`, etc.). `type` may be resolved (e.g. through `React.memo`/lazy) while `elementType` keeps the original.
- **`key`** — reconciliation identity among siblings.
- **`stateNode`** — the backing instance: a DOM node for host fibers, the class instance for class components, the `FiberRoot` for the root.
- **`return` / `child` / `sibling`** — the tree links. Not `parent`/`children` arrays: a fiber has one `child` (first child), a `sibling` (next child of the same parent), and a `return` (where to go when this subtree completes — the stack-frame return pointer). This singly-linked shape is what the work loop walks.
- **`pendingProps` / `memoizedProps`** — incoming props vs. the props used on the last committed render. Equality between them is a fast bail-out signal.
- **`memoizedState`** — for function components this is the **head of the hooks linked list** (each `useState`/`useEffect`/`useMemo` is a node); for class components it's `this.state`.
- **`updateQueue`** — pending state updates and, for host/effect fibers, the queue of side-effects to flush.
- **`flags`** (formerly `effectTag`) — a bitmask of work to perform in commit: `Placement`, `Update`, `Deletion`, `Ref`, `Passive` (effects), etc. `subtreeFlags` aggregates descendants' flags so commit can skip clean subtrees.
- **`lanes` / `childLanes`** — the priority bitmask of pending work on this fiber and its subtree (see [[lanes-and-priorities]]).
- **`alternate`** — the pointer to this fiber's counterpart in the *other* tree (double buffering).

## Two Trees, One Component

React keeps two fiber trees. The **current** tree reflects what's on screen. When an update arrives, React builds a **work-in-progress (WIP)** tree, reusing the existing fibers where possible by cloning them via `createWorkInProgress`. Each WIP fiber's `alternate` points back to its `current` counterpart and vice-versa, so React flips between them rather than allocating a whole new tree each render. On commit, the root's `current` pointer is swapped to the finished WIP tree — an O(1) flip. This is **double buffering**, the same trick graphics pipelines use; it also gives React a clean, untouched "before" tree to diff against and to fall back to if rendering is thrown away. (See [[render-and-commit-phases]].)

```text
return ←┐        current tree ⇄ WIP tree
        │        (linked via .alternate)
      [App]
        │ child
      [Header] ── sibling ──> [Main] ── sibling ──> [Footer]
```

## Mental Model And Pitfalls

Think of a fiber as a **frame in a virtual stack** that React schedules itself, and the fiber tree as a persistent scratchpad that survives across yields. Two senior-level traps follow from this:

- **Render is not commit.** A WIP fiber can be built, thrown away, and rebuilt before anything reaches the DOM. Mutating module-level state or refs *during render* corrupts this model because that work may run twice or be discarded — purity is a hard requirement, not a style guideline.
- **Identity lives in the fiber, not your closure.** Hooks read `memoizedState` off the *current* fiber via a module-level dispatcher pointer; this is why hook order must be stable and why a remounted component (new fiber, new `stateNode`) resets all state even if the element looks identical. `key` changes force exactly that swap.

Fiber is the substrate; everything React 19 does — concurrency, transitions, Suspense — is built on the fact that this stack is now data you can pause, prioritize, and replay.

## See Also

- [[work-loop-and-scheduling]]
- [[lanes-and-priorities]]
- [[render-and-commit-phases]]
- [[concurrent-rendering]]
- [[10-frameworks-and-stacks/react-native/architecture/rendering|RN Rendering]]
- [[hooks-internals]] — the per-fiber hook list
