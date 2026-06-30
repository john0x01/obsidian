# JavaScript Runtime

JavaScript the language only specifies a single-threaded execution model and a "job" mechanism; everything that makes it *do* anything — timers, DOM, network, file I/O — lives in the **host environment** (browser, Node, Deno). The "runtime" is the union of an **engine** (executes ECMAScript) plus the **host APIs and the event loop** that drive it. Confusing the two is the single most common source of mental-model errors about async behaviour.

## Engine vs Runtime/Host

- **Engine** (V8, SpiderMonkey, JavaScriptCore): owns the heap, the call stack, parsing/compilation, GC, and the *microtask queue* (the ECMAScript "Job" queue). The engine knows nothing about `setTimeout` or `fetch`.
- **Host/runtime**: embeds the engine and supplies global objects (`window`, `process`), Web/Node APIs, the **task (macrotask) queues**, and the **event loop** that decides *when* the engine runs. The HTML spec defines the browser event loop; libuv defines Node's.

```
+-------------------- Host (browser / Node) --------------------+
|  Web/Node APIs   Task queues   Event loop   (rendering, I/O)  |
|        +------------------- Engine (V8) -------------------+   |
|        |  Parser -> Bytecode/JIT   Call stack   Heap   GC  |   |
|        |  Microtask (Job) queue                            |   |
|        +---------------------------------------------------+   |
+---------------------------------------------------------------+
```

## Call Stack and Execution Contexts

The engine runs synchronous code on one **call stack**. Each call pushes an **execution context** (the spec's *running execution context*) and returns pop it. A context is not just "a frame with locals" — per ECMAScript it bundles:

- **LexicalEnvironment** — resolves `let`/`const`/`class` and `function` declarations; a chain of *Environment Records* linked by `[[OuterEnv]]` to the enclosing scope. This chain is what makes [[closures]] work: a returned function keeps a reference to the environment record it closed over, so that record (and only the bindings it needs, after escape analysis) stays alive on the heap.
- **VariableEnvironment** — the record `var` and `function` declarations bind into. At the top of a context, declarations are *instantiated* before any code runs — the mechanical truth behind "hoisting" and the `let` Temporal Dead Zone (the binding exists but is marked uninitialized).
- **`this` binding** — resolved per call type: method call (receiver), plain call (`undefined` in strict mode / global object otherwise), `new` (fresh object), or lexically inherited by arrow functions, which capture `this` from the *defining* context and have no binding of their own.

Stack overflow is simply this stack hitting its bound; it is per-engine, not specified by ECMAScript. Proper tail calls (which would reuse a frame) are specced but only JavaScriptCore ships them.

## Heap

Objects, closures, and arrays live in the **heap** — an unstructured region the engine allocates from and the [[garbage-collector]] reclaims. Primitives are conceptually values, but engines box, intern, and use NaN-tagging / Smi (small-integer) representations under the hood. The stack holds frames and references; the heap holds the actual mutable graph. Reachability *from* the stack and globals (the GC roots) is what keeps heap objects alive.

## The Event Loop, Macrotasks, and Microtasks

When the stack empties, the host's event loop picks the next unit of work. Two distinct queues, with strict ordering:

- **Macrotask / task queue**: one task per loop turn — a timer callback, an I/O completion, a UI event, a `MessageChannel` message. `setTimeout(fn, 0)` enqueues a *task* (clamped to ~4ms after nesting in browsers).
- **Microtask / Job queue**: Promise reactions, `queueMicrotask`, `MutationObserver`, `await` continuations. Drained **completely** — including microtasks enqueued *by* microtasks — after each macrotask and after each callback, before yielding.

```
loop turn:
  run ONE macrotask  -> stack runs to empty
  drain ALL microtasks (including newly added ones)
  (browser) maybe render
  -> next macrotask
```

This is why `Promise.resolve().then(...)` always runs before a `setTimeout(...,0)` queued in the same tick, and why an infinite microtask chain can starve rendering and tasks entirely. `await` is sugar: it suspends the async function and schedules the continuation as a microtask when the awaited promise settles.

## How Rendering Interleaves (Browsers)

The browser aims for ~60fps, so it tries to render once per ~16.6ms *between* macrotasks, never mid-task. The sequence per frame: run a task, drain microtasks, then run the **rendering steps** — `requestAnimationFrame` callbacks fire here (before layout/paint), style/layout/paint happen, then `IntersectionObserver` callbacks. `requestIdleCallback` runs only if spare time remains. Long synchronous tasks (>50ms) block this entire pipeline: the page can't paint or respond, the classic "jank"/"unresponsive" pathology. Node has no rendering step; its loop has ordered *phases* (timers, pending, poll, check for `setImmediate`, close) with microtasks (and `process.nextTick`, which jumps ahead of even promise microtasks) drained between callbacks.

**Senior pitfalls**: assuming `setTimeout(0)` is "immediate" (it yields a full task + render); blocking the loop with sync CPU work instead of chunking or offloading to a Worker; microtask starvation from recursive `.then`; and conflating Node's `nextTick`/`setImmediate` ordering with the browser model.

## See Also
- [[garbage-collector]] — heap reclamation for objects on this heap
- [[v8-architecture]] — the engine that owns the stack and heap
- [[closures]] — environment records kept alive past return
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]] — the host loop driving the engine
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]] — microtask-scheduled reactions
- [[03-computer-systems/concurrency-and-parallelism/async-and-await|Async and Await]] — await as microtask continuation
- [[10-frameworks-and-stacks/node-js/architecture/libuv-and-event-loop|Node Event Loop (libuv)]] — the same loop in Node
