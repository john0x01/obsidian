# Node.js Runtime Architecture

Node is not a language and not "JavaScript with extra functions" — it is an **embedder**: a C++ program that links V8 as a library, drives it with libuv's event loop, and exposes operating-system capabilities to JS through a thin binding layer. Understanding Node means understanding that composition, because almost every confusing async behaviour is an emergent property of how these independent C/C++ subsystems are stitched together, not a property of the JS spec.

## The Four Layers

```
+---------------------------------------------------------------+
|  Core JS libraries (lib/*.js): fs, http, stream, events, ...  |  JavaScript
+---------------------------------------------------------------+
|  C++ bindings (src/*.cc): node::Buffer, fs, crypto, http2 ... |  glue
+----------------------+----------------------------------------+
|  V8 (engine)         |  libuv (event loop, thread pool, I/O)  |  C/C++ deps
+----------------------+----------------------------------------+
|  OS: epoll/kqueue/IOCP, threads, sockets, syscalls            |
+---------------------------------------------------------------+
```

- **V8** owns ECMAScript: parse, Ignition bytecode, Maglev/TurboFan JIT, the heap, GC, and the *microtask* (Job) queue. V8 has no concept of timers, sockets, or files.
- **libuv** is a C library that provides the **event loop**, a **thread pool**, async filesystem and DNS, timers, child processes, signals, and a cross-platform socket/pipe abstraction. It is what makes Node *do I/O*. Node is libuv's most famous embedder, but libuv is independent.
- **C++ bindings** are the bridge: they register native functions on "internal binding" objects, marshal V8 values to C types, submit work to libuv, and convert libuv completions back into V8 calls.
- **Core JS** (`lib/`) is ordinary JavaScript shipped inside the binary as a snapshot. `fs.readFile` is a JS function that ultimately calls an internal binding.

Also present and easy to forget: **c-ares** (async DNS for `resolve*`), **OpenSSL/BoringSSL** (TLS, crypto), **zlib**, **llhttp**, **ngtcp2/nghttp2/nghttp3**, and **simdjson**-style fast paths. Node's binary is the static composition of all of these.

## How a JS Call Reaches Native Async and Back

Trace `fs.readFile('a', cb)`:

1. JS `fs.readFile` validates args, allocates a `FSReqCallback` request object, and calls the C++ binding `binding.read(...)`.
2. The binding fills a `uv_fs_t` request and calls `uv_fs_read(loop, req, ..., uv__fs_done)`. Because a callback was supplied, libuv hands the work to the **thread pool** (filesystem ops are not natively async on most OSes).
3. The originating JS call **returns immediately**. The call stack unwinds; the main thread goes back to spinning the event loop.
4. A pool thread performs the blocking `pread` syscall. On completion it posts the result back to the loop via libuv's internal async/pipe wakeup mechanism so the loop's `poll` phase observes it.
5. In the loop's next iteration the completion is dispatched, the C++ callback (`uv__fs_done` → `AfterRead`) runs **on the main thread**, builds V8 values (a `Buffer`), and invokes your JS callback via `MakeCallback` — which also drains `process.nextTick` and the microtask queue afterward.

The crucial invariant: **all JavaScript runs on one thread**. Pool threads never touch V8; they only do syscalls and signal completion. This is why Node has no data races *in JS* despite using OS threads underneath.

## Single-Threaded JS, Multi-Threaded I/O

"Node is single-threaded" is true only of *your JavaScript*. The process has many threads: the main loop thread, the (default 4) libuv pool workers, V8's background compiler/GC helper threads, and any `Worker` threads you spawn (each its own V8 isolate + loop). The single-threaded *programming model* is the asset: no locks around your application state, deterministic mutation, simple reasoning. The liability is symmetric — one long synchronous function (a tight loop, `JSON.parse` of 50MB, sync crypto) blocks *everything*, because there is no other thread to run the loop. Concurrency in Node is **cooperative**: it exists only at the boundaries where you yield by awaiting or returning to the loop.

## Contrast With the Browser Runtime

Same engine (V8 in Chromium), different host. The browser's host is defined by the HTML spec; its event loop has a **rendering pipeline** (`requestAnimationFrame`, style/layout/paint) and task sources tied to the DOM, and it deliberately throttles `setTimeout`. Node's host is libuv; its loop has **ordered phases** (timers → pending → poll → check → close) and no rendering step, plus two extras the browser lacks: `setImmediate` (the *check* phase) and `process.nextTick` (drained before microtasks). The browser sandboxes away the filesystem and raw sockets; Node exposes them directly. Same `Promise` microtask semantics, materially different macrotask scheduling — which is exactly why ordering snippets that "work" in DevTools mislead people about Node.

## Senior Pitfalls

- Treating Node as "JS + a stdlib" and being surprised that CPU work freezes I/O — the loop is your code's *only* scheduler.
- Assuming `fs` async means OS-async; it is usually **pool-async** (see [[thread-pool]]), so it competes with crypto/zlib for 4 threads.
- Forgetting `Worker` threads are separate isolates: no shared object graph, only `SharedArrayBuffer`/message-passing.

## See Also
- [[libuv-and-event-loop]] — the loop that drives V8
- [[thread-pool]] — where pool-async work actually runs
- [[process-lifecycle]] — bootstrap and the process object
- [[01-programming-foundations/languages/javascript/engine-internals/v8-architecture|V8 Architecture]] — the embedded engine
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]] — engine vs host distinction
- [[03-computer-systems/operating-systems/io-models|I/O Models]] — readiness vs completion underneath libuv
