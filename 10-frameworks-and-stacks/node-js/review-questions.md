# Review Questions

A senior-level self-test for Node.js (target: Node 22 LTS). These are not lookups — answer them out loud or in writing, and where there is a trade-off, argue both sides before stating your default.

## Architecture & Event Loop

- Walk through libuv's loop phases in order. What runs in each, and where do `setTimeout`, `setImmediate`, and I/O callbacks actually fire?
- Where do the microtask and `process.nextTick` queues drain relative to the phases, and which drains first? Why does the ordering matter for fairness?
- Explain why `setImmediate` and `setTimeout(fn, 0)` can fire in either order at the top level but have deterministic order inside an I/O callback.
- What exactly is "blocking the event loop"? Give three distinct mechanisms (CPU, sync I/O, microtask flood) and how each starves a *different* part of the loop.
- How does Node bridge the single-threaded JS world to libuv's thread pool? What work uses the thread pool, and what uses the OS kernel's async primitives instead?
- What is `UV_THREADPOOL_SIZE`, what defaults to it, and when would raising or lowering it help or hurt?
- How does the event loop interact with `worker_threads`? Does each worker get its own loop, and what is shared vs isolated between them?
- Distinguish event loop *delay* from event loop *utilization*. Which is a leading indicator, which is lagging, and which would you alert on vs autoscale on?
- Why is a recursive `process.nextTick` more dangerous than a recursive `setImmediate`?
- How does Node decide the process exits? What keeps the loop alive, and what does `ref()`/`unref()` change?

## Modules

- Contrast CommonJS and ESM resolution: how does each locate a specifier, and where do `node_modules` walking, conditional `exports`, and `imports` map fields come in?
- Explain CJS/ESM interop: why is named-export interop from CJS into ESM limited, and what does the default-export-as-`module.exports` shim do?
- What is the live-bindings semantics of ESM, and how does it differ from CJS's snapshot-at-`require` behavior? Construct a case where this difference is observable.
- How does the module cache work in CJS vs ESM? Can you reliably bust either, and what breaks if you do?
- What is the role of the `"type"` field, file extensions (`.mjs`/`.cjs`), and `package.json` `exports`/`imports` in determining module format and entry points?
- Explain top-level `await` in ESM: what does it do to the module graph's evaluation order and to anything that `import`s the module?
- How do `require(esm)` (the synchronous ESM-from-CJS capability) and dynamic `import()` differ in what they can load and when?
- What does a customization hook (the `module.register`/loader hooks API) let you intercept, and what's a legitimate use vs an abuse?

## Streams & I/O

- Explain backpressure in Node streams. What does `.write()` returning `false` mean, who is responsible for pausing, and how does `pipe`/`pipeline` handle it for you?
- Contrast flowing and paused modes for Readable streams. What event or method moves between them, and what's the data-loss risk?
- Why is `stream.pipeline` (or its promise form) preferred over manual `.pipe()` chains? What does it guarantee about error propagation and resource cleanup?
- How do object-mode streams change `highWaterMark` semantics?
- What is the relationship between async iterators (`for await...of`) and Readable streams? What are the backpressure and error-handling implications of consuming a stream that way?
- When would you reach for a Transform stream vs simply mapping in an async iterator? What does each cost?
- Explain how a single `error` on one stream in a pipe chain can leak file descriptors or sockets if not handled, and how `pipeline` prevents it.

## Concurrency

- Node is "single-threaded" — so what does `worker_threads` actually give you, and when is it the right tool vs `cluster` vs separate processes?
- How is memory shared between worker threads? Explain `SharedArrayBuffer`, `Atomics`, and the structured-clone cost of `postMessage`.
- When is `cluster` (or running N processes behind a load balancer) preferable to one process with a worker pool? What does each model do for fault isolation and CPU utilization?
- Describe a sound pattern for a CPU-bound worker pool: how do you size it, queue work, and avoid head-of-line blocking?
- What concurrency hazards survive even in single-threaded Node (interleaving at `await` points, shared mutable module state, check-then-act races on external resources)?
- How would you bound concurrency of N outbound requests so you neither serialize them nor exhaust sockets/file descriptors? Name the primitives.
- What does `AbortController`/`AbortSignal` give you for cancellation, and what are its limits (cooperative, not preemptive)?

## Performance

- Given "the service is slow," what is your first question, and how do you decide whether you're CPU-bound, GC-bound, or latency/I-O-bound *before* picking a tool?
- How do you read a CPU flame graph? What does width encode, why isn't it a timeline, and why can it look idle precisely when you're latency-bound?
- Compare `--prof` + tick processor, an inspector `.cpuprofile`, and trace events. When does each one earn its keep?
- Distinguish RSS, heapTotal, heapUsed, and external in `process.memoryUsage()`. Which one rising points at which class of leak?
- Describe the three-snapshot diff technique for finding a leak. What are you sorting by, and what is the "smoking gun"?
- Name the most common sources of Node memory leaks and the defense for each (closures over large scope, unbounded module-level caches, un-removed EventEmitter listeners, orphan timers).
- When and why would you set `--max-old-space-size`, and how do you choose the value in a memory-limited container?
- What does `--max-semi-space-size` tune, and what observable behavior would make you reach for it?
- How do hidden classes / inline caches in V8 make "monomorphic" code fast, and what coding patterns deoptimize a hot function?

## Security

- State Node's core trust assumption and contrast it with a browser's. Why does this make supply chain the dominant risk?
- Summarize the Node threat model: give two examples of vulnerabilities that are in scope for core to fix and two that are explicitly your responsibility.
- What does the `--permission` model enforce, what does it *not* protect (network, addons), and why is it not a sandbox for untrusted code?
- Why is `child_process.exec` dangerous with any user input, and what is the precise fix using `execFile`/`spawn`?
- Explain prototype pollution end to end: the sink, how it escalates, and three defenses you'd apply.
- What makes a regex vulnerable to ReDoS, why is the impact disproportionate in Node specifically, and how do you mitigate it for untrusted input?
- How do you correctly prevent path traversal when serving files from a directory? Why is rejecting `..` in the raw string insufficient?
- What is the difference between `vm` and a real sandbox, and how would you actually run untrusted code safely?
- What concrete supply-chain controls would you put in a CI pipeline (lockfiles, `npm ci`, `--ignore-scripts`, audit, provenance), and what residual risk remains?
