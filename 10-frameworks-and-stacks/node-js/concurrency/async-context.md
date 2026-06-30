# Async Context

In a single-threaded, callback-driven runtime there is no thread-local storage to stash "the current request" in — by the time a database callback fires, the stack frame that started the request is long gone. `async_hooks` and its high-level consumer `AsyncLocalStorage` solve exactly this: they let you **propagate ambient context (a request ID, a trace span, a tenant, a user) across `await`s, timers, and callbacks** without threading an argument through every function. This is Node's answer to continuation-local storage (CLS).

## The Execution-Context Model

The enabling idea is that every asynchronous operation in Node lives inside an **async resource** — a Promise, a `setTimeout`, an incoming TCP socket, an `fs` request. The runtime assigns each resource an `asyncId` and, crucially, records the `asyncId` of the resource that was *active when it was created* — its **triggerAsyncId**. That parent-child relationship forms an *async call graph*: a tree of "who scheduled whom," distinct from the synchronous call stack which is destroyed at every `await`.

`async_hooks.createHook({ init, before, after, destroy })` lets you observe this graph:

- **`init(asyncId, type, triggerAsyncId, resource)`** — a new async resource is born; you learn its type and parent.
- **`before` / `after`** — the runtime is about to run / has finished running that resource's callback. Between `before` and `after`, `executionAsyncId()` returns this resource.
- **`destroy`** — the resource is gone (eligible for GC).

The runtime maintains, for the current synchronous run, an *execution context* identified by `executionAsyncId()`. As callbacks fire, it swaps the active id in and out via `before`/`after`. Context propagation is then conceptually simple: store a value keyed by async-resource lineage, and when a callback runs, look it up by walking to the right entry. `AsyncLocalStorage` does this for you and is the API you should actually use — raw `async_hooks` is low-level, easy to misuse, and its hook API is marked experimental.

## AsyncLocalStorage in Practice

```js
import { AsyncLocalStorage } from 'node:async_hooks';
const als = new AsyncLocalStorage();

// At the edge (e.g. HTTP middleware):
als.run({ requestId: crypto.randomUUID() }, () => {
  handle(req, res); // everything async spawned from here sees the store
});

// Anywhere deep in the call graph, no argument threading:
function log(msg) {
  const store = als.getStore();          // the SAME object set above
  logger.info({ requestId: store?.requestId }, msg);
}
```

`run(store, callback)` establishes a store for the duration of `callback` *and every async operation transitively triggered by it*. `getStore()` returns whatever store is bound to the currently-executing async context, or `undefined` outside any `run`. Because propagation follows the *async resource* lineage rather than the lost stack, the store survives `await`, `.then`, `setTimeout`, event-emitter callbacks, and stream events — all the boundaries a plain variable could not cross. `enterWith(store)` sets the store for the rest of the *current* execution without nesting, but it is sharp-edged (it mutates the current context and can leak into siblings); prefer `run`.

This is the machinery behind request-scoped logging, distributed tracing (OpenTelemetry's Node context manager is built on it), per-request DB transactions, and multi-tenant scoping.

## Performance Considerations

Context tracking is not free, and the cost model has changed over Node's life. Historically, enabling any `async_hooks` hook forced V8 to invoke `PromiseHook` on *every* promise creation/resolution, imposing a measurable tax across the whole process even on code that never touches your store. Modern Node (V8's native `AsyncContextFrame`, which `AsyncLocalStorage` uses by default in Node 22-class runtimes) makes propagation far cheaper by carrying the context on the continuation rather than diffing the resource graph — closer to "pay for what you use." Still:

- The overhead scales with promise/async-op volume, so the hottest async loops feel it most.
- Putting large objects in the store keeps them reachable for the entire async lifetime — a subtle way to inflate retained memory and pressure the GC.
- Many `createHook` consumers active at once compound the cost; libraries should share one store, not each install hooks.

## Pitfalls and Misconceptions

- **It is not thread-local.** Each Worker Thread has its own `AsyncLocalStorage` instances and stores; context does not cross the postMessage boundary. You must re-establish context inside a Worker.
- **Context can leak across logical requests** if you reuse a long-lived async resource (a pooled connection, a shared emitter) created *outside* any `run`. The fix is to wrap the per-request work in `run`, not the shared resource's construction.
- **Manual queues and custom thenables break propagation** if they detach from Node's async resources — e.g., resolving promises stored in a hand-rolled scheduler can lose the context. Most native primitives and `await` preserve it; exotic control flow may not.
- **`getStore()` returning `undefined`** almost always means code ran *before* `run` wrapped it, or on a callback whose triggering resource predates the `run` — i.e., an init-order bug, not a Node bug.
- **Do not use it as a god-object.** It is for cross-cutting *context*, not for smuggling dependencies that should be passed explicitly; overuse hides data flow and makes testing harder.

## Philosophy

`AsyncLocalStorage` restores something object-oriented languages take for granted — implicit per-request scope — to an environment where the call stack is ephemeral. It trades a little runtime magic and overhead for enormous ergonomic wins in observability. The senior stance: use it for genuinely cross-cutting concerns (tracing, request IDs, tenancy) where explicit threading would pollute every signature, and pass everything else by argument.

## See Also

- [[worker-threads]]
- [[cluster-and-child-processes]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JavaScript Runtime]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
- [[03-computer-systems/concurrency-and-parallelism/futures-and-promises|Futures and Promises]]
