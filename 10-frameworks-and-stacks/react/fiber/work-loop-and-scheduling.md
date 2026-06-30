# Work Loop And Scheduling

The **work loop** is the engine that drives reconciliation over the fiber tree. Instead of recursing down the component tree on the native call stack, React iterates a loop that processes one fiber at a time, which is what makes rendering **incremental** and **interruptible**. Sitting underneath it is the **Scheduler** — a separate package (`scheduler`) that decides *when* the loop is allowed to run so React can cooperatively yield the main thread back to the browser.

## The Two-Pass Walk: beginWork / completeWork

The render phase is a depth-first traversal expressed as a loop over a `workInProgress` pointer:

- **`beginWork(fiber)`** — descends. It reconciles the fiber's children (diffing elements, creating/cloning child fibers) and returns the first child as the next unit. This is where your function component is actually *called* and hooks run.
- **`completeWork(fiber)`** — ascends. When a fiber has no child to descend into, React completes it: for host components it creates/prepares the DOM instance and bubbles `flags`/`subtreeFlags` up via the `return` pointer; then it moves to the `sibling`, or completes the parent if there's none.

```text
beginWork ↓        completeWork ↑
   App                 App
  /                   ↑
 Header → Main      Header → Main
```

The loop, conceptually:

```js
function workLoopConcurrent() {
  while (workInProgress !== null && !shouldYield()) {
    workInProgress = performUnitOfWork(workInProgress); // begin, then complete-or-sibling
  }
}
```

The single line that distinguishes concurrent from legacy is `!shouldYield()`. The synchronous variant (`workLoopSync`) omits it and runs to completion.

## Cooperative Yielding And Time Slicing

`shouldYield()` asks the Scheduler whether the current time slice (a frame budget, on the order of ~5ms) is exhausted. If so, the loop returns control to the browser so it can paint, run layout, handle input, and fire other tasks — then React resumes the loop on the next scheduled tick, picking up at the saved `workInProgress`. Because all per-fiber state lives in the heap-allocated fiber tree (see [[fiber-architecture]]), pausing is just "stop the loop"; no stack needs unwinding. This is **time slicing**: a long render is sliced across many macrotasks instead of monopolizing one.

Crucially, only the **render phase** yields. Once React enters **commit**, it runs synchronously and atomically — you never see a half-applied DOM (see [[render-and-commit-phases]]).

## The Scheduler Package

`scheduler` is intentionally framework-agnostic (it's a generic priority-queue task runner React happens to consume). Key mechanics:

- **MessageChannel, not setTimeout.** To hand control back and regain it on the next macrotask, the Scheduler posts a message on a `MessageChannel`'s port; the `onmessage` callback runs as a fresh macrotask after the browser has had its turn. `setTimeout(fn, 0)` is avoided because browsers clamp nested timeouts to ~4ms, wasting budget. `MessageChannel` reschedules with near-zero delay. (`requestIdleCallback` was rejected for inconsistent cross-browser timing.)
- **A priority-ordered task queue.** Tasks carry priorities — `ImmediatePriority`, `UserBlockingPriority`, `NormalPriority`, `LowPriority`, `IdlePriority` — implemented as a min-heap keyed by an **expiration time**. Higher priority ⇒ sooner expiration.
- **Expiration / starvation avoidance.** Each task gets a deadline derived from its priority. While a task hasn't expired, the Scheduler may keep yielding to the browser. Once `currentTime >= expirationTime`, the task is treated as expired and run *synchronously without further yielding*, guaranteeing even low-priority work eventually completes. React's lane model layers its own expiration on top of this (see [[lanes-and-priorities]]).

Note the two priority systems: the Scheduler's coarse priorities govern *task execution timing*, while React's **lanes** govern *which updates belong to a given render*. React translates lanes into a Scheduler priority when it asks to be called back.

## Versus The Old Stack Reconciler

The pre-16 reconciler was a synchronous recursive walk on the native stack. Once an update began, it ran to the end — no interruption, no reprioritization, no abandoning stale work. A heavy subtree update blocked input handling and animation, producing visible jank. The work loop fixes this structurally: rendering becomes a resumable loop over data, and the Scheduler decides how to interleave it with the browser's own work.

## Senior Pitfalls

- **Yielding ≠ parallelism.** It's all on the single main thread; time slicing reduces *blocking*, not total CPU. A genuinely expensive render is still expensive — memoize or split it.
- **Restarted renders run `beginWork` again.** Because an interrupted higher-priority update can discard the WIP tree, your render function (and `useMemo` factories) can run more than once before a commit. Effects don't — they're deferred to commit — which is exactly why side effects must live in effects, not in render bodies.
- **`shouldYield` is time-based, not work-based.** A single fiber whose `beginWork` is itself slow (e.g. a giant unmemoized list item) can't be sliced mid-fiber; the granularity of interruption is one fiber.

## See Also

- [[fiber-architecture]]
- [[lanes-and-priorities]]
- [[render-and-commit-phases]]
- [[concurrent-rendering]]
- [[10-frameworks-and-stacks/react-native/architecture/threading-model|RN Threading Model]]
