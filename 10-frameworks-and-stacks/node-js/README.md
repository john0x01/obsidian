# Node.js

Deep notes on the Node.js runtime — its architecture (V8 + libuv), the event loop, modules, streams, concurrency, and production concerns. Targets Node.js 22 LTS. For mastery, not API lookup.

[← Back to Frameworks & Stacks](../README.md)

## Architecture
- [Runtime Architecture](architecture/runtime-architecture.md) — V8 + libuv + bindings; how a JS call reaches async I/O
- [libuv & the Event Loop](architecture/libuv-and-event-loop.md) — the loop phases
- [Timers & Microtasks](architecture/timers-and-microtasks.md) — `nextTick` vs microtasks vs `setTimeout`/`setImmediate`
- [The Thread Pool](architecture/thread-pool.md) — what uses it, what doesn't
- [Process Lifecycle](architecture/process-lifecycle.md) — bootstrap, signals, graceful shutdown

## Modules
- [CommonJS Internals](modules/commonjs-internals.md) — the wrapper, cache, circular deps
- [ESM in Node](modules/esm-in-node.md) — interop, dual packages, the async loader
- [Module Resolution](modules/module-resolution.md) — `node_modules`, `exports`/`imports` maps

## Streams
- [Streams Model](streams/streams-model.md) — Readable/Writable/Duplex/Transform, modes
- [Backpressure](streams/backpressure.md) — `highWaterMark`, `drain`, `pipeline`

## I/O
- [Buffers](io/buffers.md) — Buffer/TypedArray, the allocation pool, `allocUnsafe`
- [File System & Networking](io/file-system-and-networking.md)

## Concurrency
- [Worker Threads](concurrency/worker-threads.md) — separate isolates, message passing
- [Cluster & Child Processes](concurrency/cluster-and-child-processes.md)
- [Async Context](concurrency/async-context.md) — `async_hooks`, `AsyncLocalStorage`

## Native
- [Native Addons & N-API](native/native-addons-and-napi.md)

## Performance
- [Profiling & Diagnostics](performance/profiling-and-diagnostics.md)
- [Memory & GC](performance/memory-and-gc.md) — heap snapshots, leaks, tuning
- [Event Loop Monitoring](performance/event-loop-monitoring.md) — lag & utilization

## Security
- [Security Model](security/security-model.md) — the permission model, common vuln classes

## Self-Test
- [Review Questions](review-questions.md)
