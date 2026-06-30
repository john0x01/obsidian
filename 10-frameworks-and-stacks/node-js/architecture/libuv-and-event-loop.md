# libuv and the Event Loop

libuv is the C library that gives Node its asynchrony: a single-threaded **event loop** built on the OS's best readiness primitive, plus a thread pool for work the OS cannot do asynchronously. The event loop is not a queue — it is a **state machine that walks a fixed sequence of phases**, and almost every "why did this callback run when it did?" question is answered by knowing which phase owns the callback.

## The Phases

Each full traversal is called a **loop iteration** ("a tick" in casual usage, though `process.nextTick` is unrelated to this tick). libuv runs phases in this order:

```
   +-> timers           setTimeout / setInterval whose time has elapsed
   |   pending callbacks deferred I/O callbacks (e.g. some TCP errors)
   |   idle, prepare     internal libuv bookkeeping
   |   poll  -----------  wait for & process I/O; may BLOCK here
   |   check             setImmediate callbacks
   +-- close callbacks   'close' events (socket.on('close'), etc.)
```

Between *each* callback in *every* phase, Node drains the `process.nextTick` queue and then the V8 microtask (Promise) queue — that interleaving is Node-specific and covered in [[timers-and-microtasks]]. libuv itself knows nothing about microtasks; Node injects those drains at the binding boundary.

- **timers**: runs timer callbacks whose threshold has expired. The threshold is a *minimum*, not a guarantee — a busy poll phase or long callback pushes timers late.
- **pending callbacks**: a small set of callbacks libuv defers from the previous iteration (e.g. certain `ECONNREFUSED`-style errors on some platforms).
- **idle / prepare**: internal only; no user-facing API beyond `setImmediate` indirectly. `prepare` handles run right before polling.
- **poll**: the heart of the loop. Two jobs: execute completed I/O callbacks, and **decide how long to block** waiting for new I/O.
- **check**: `setImmediate` callbacks. Designed to run *right after* poll, so "do this after I/O drains, before timers" maps to `setImmediate`.
- **close callbacks**: cleanup callbacks for handles closed abruptly.

## The Poll Phase and Blocking

`poll` is where the process actually sleeps. Its blocking decision:

- If there are `setImmediate` callbacks pending (`check` has work), poll does **not** block — it returns so `check` can run immediately.
- Otherwise, if there are timers, poll blocks **only until the nearest timer is due**, so timers fire on time.
- If there is neither, poll blocks **indefinitely** on the OS until an fd becomes ready — this is why an idle server consumes ~0% CPU. It is genuinely asleep in a syscall, not spinning.

When poll wakes (an fd is readable/writable, or a timer is due, or the pool posted a completion), it runs the ready I/O callbacks, then advances to `check`. If too much time accrues, libuv caps how long it stays in poll to avoid starving timers.

## How libuv Abstracts the OS

The poll phase wraps each platform's readiness/completion mechanism behind one API (`uv__io_poll`):

- **Linux** → `epoll` (edge-ish, scalable readiness; moving toward `io_uring` for some fs ops in newer libuv).
- **macOS/BSD** → `kqueue`.
- **Windows** → **IOCP**, a *completion* model rather than a readiness model.

This is a real impedance mismatch. epoll/kqueue are **reactor** (readiness: "the socket is now readable, you go read it"); IOCP is **proactor** (completion: "your read finished, here's the data"). libuv presents a uniform *completion-style* callback API on top of both, emulating completion over readiness on Unix. Network sockets ride this fd-polling path directly — they never touch the thread pool. Operations the OS *can't* do via fd readiness (most filesystem calls, `getaddrinfo`, CPU-bound crypto/zlib) are offloaded to the pool, which posts results back so poll observes them like any other readiness event.

## What "a Tick" Means

Ambiguous term, two senses:

1. **A libuv loop iteration** — one full pass through the phases above. `setImmediate` schedules "next iteration, check phase."
2. **`process.nextTick`** — *not* a loop iteration at all; it runs before the loop continues, draining after the current operation. The name is historical and misleading.

The loop exits when there are no more active **handles** (servers, sockets, timers) or **requests** keeping a reference. `ref`/`unref` controls whether a handle counts toward keeping the loop alive (see [[process-lifecycle]]).

## Senior Pitfalls

- Believing phases run "in parallel" — they are strictly sequential on one thread; only the pool and OS work concurrently.
- Expecting `setTimeout(fn, 0)` and `setImmediate(fn)` to have a stable order at the top level — they don't (timer threshold rounding races the loop start); *inside an I/O callback* `setImmediate` always wins because you're already past poll, so `check` is next.
- Assuming a slow disk read blocks the loop — it blocks a *pool thread*; the loop keeps polling. A slow *synchronous* call blocks the loop.

## See Also
- [[timers-and-microtasks]] — nextTick/microtask drains between phase callbacks
- [[thread-pool]] — work poll cannot do via fd readiness
- [[runtime-architecture]] — where libuv sits in Node's composition
- [[process-lifecycle]] — handles, ref/unref, and loop exit
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]] — reactor pattern in the abstract
- [[03-computer-systems/operating-systems/io-models|I/O Models]] — epoll/kqueue/IOCP, reactor vs proactor
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime & Event Loop]] — the browser/engine model
