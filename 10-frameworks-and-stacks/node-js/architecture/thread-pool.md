# The libuv Thread Pool

The libuv thread pool is the hidden multi-threaded engine behind Node's "asynchronous" filesystem, DNS lookup, crypto, and compression. It exists because not everything the OS offers can be done through fd readiness (epoll/kqueue) — some operations are inherently blocking syscalls or CPU-bound work, and the only way to keep the single-threaded event loop responsive is to hand them to background threads. Knowing exactly *what* uses the pool, and that its default size is **4**, prevents a whole class of mysterious latency bugs.

## What It Is

A fixed-size set of worker threads created lazily on first use, managed by libuv's `uv__work_submit`. The event loop submits a work request (`uv_work_t` semantics) with a *work* function (runs on a pool thread) and an *after-work* function (runs back on the loop thread). The pool is a classic bounded worker pool fronted by a queue:

```
loop thread                    pool (default 4 threads)
  submit work  --> [ queue ] --> thread runs blocking syscall / CPU work
                                       |
  after-work cb <-- post done <--------+   (loop sees it in poll phase)
```

Work functions run **truly in parallel** on separate OS threads, but they **never touch V8** — they do raw C work (a `pread`, an OpenSSL `PBKDF2`, a zlib deflate) and hand the result back. The after-work callback, which builds JS values and calls your callback, runs on the single loop thread. So parallelism is real for the *native* portion only.

## What Uses the Pool

- **`fs.*` async operations** — almost all of them. Filesystem syscalls are not reliably async on Unix, so libuv offloads them. (`io_uring`, where libuv enables it, can change this for some ops, but assume pool by default.)
- **`dns.lookup`** — uses the OS resolver via blocking `getaddrinfo` on a pool thread. **Crucial distinction:** `dns.resolve*` / `dns.lookupService` use **c-ares** and go over the network directly — *not* the pool.
- **`crypto`** async calls — `pbkdf2`, `scrypt`, `randomBytes`/`randomFill`, key generation, etc.
- **`zlib` / Brotli** async compression and decompression.
- A few others (e.g. `fs.realpath` style helpers, some `getnameinfo`).

## What Does NOT Use the Pool

- **Network I/O** — TCP/UDP sockets, HTTP servers/clients, pipes. These are pollable fds handled natively by epoll/kqueue/IOCP in the **poll** phase. A thousand concurrent sockets cost zero pool threads. This is the single most important fact: Node scales to massive connection counts precisely *because* networking bypasses the pool.
- **Timers, `setImmediate`, `process.nextTick`, Promise microtasks** — pure loop/queue constructs.
- **`*Sync` calls** (`readFileSync`, `pbkdf2Sync`) — these block the loop thread directly; the pool is irrelevant.

## Sizing: UV_THREADPOOL_SIZE

The default is **4**. Override via the environment variable `UV_THREADPOOL_SIZE`, read **once at startup** when the pool initialises — setting `process.env.UV_THREADPOOL_SIZE` after the first pool use has no effect. The historical max was 128; treat the variable as the knob and verify against your Node version rather than relying on a remembered ceiling.

```bash
UV_THREADPOOL_SIZE=16 node server.js
```

Sizing heuristic: the pool is for blocking/CPU work, so the useful size relates to how many such operations you want concurrent and how many cores you have. More threads than cores helps for **I/O-bound** pool work (threads block on disk), but for **CPU-bound** pool work (crypto, zlib) oversubscription past core count just adds context-switch overhead. There is no universal number — measure.

## Blocking the Pool ("Pool Starvation")

The pool is **shared** across `fs`, `dns.lookup`, `crypto`, and `zlib`. With the default 4 threads, four concurrent slow `scrypt` calls occupy the entire pool; a fifth — and *any* `fs.readFile`, and *any* `dns.lookup` — queues behind them. Symptom: file reads and hostname resolution suddenly develop multi-hundred-millisecond latency under crypto load, with the event loop itself looking healthy (low loop lag) because the loop is fine — it's the pool that's saturated.

Diagnosing this is non-obvious because event-loop-lag metrics won't show it. Watch for: (a) latency on pool-backed ops that correlates with crypto/zlib volume; (b) improvement when you raise `UV_THREADPOOL_SIZE` (a strong tell). Mitigations: raise the pool size, move heavy CPU work to a [[03-computer-systems/concurrency-and-parallelism/thread-pools|dedicated]] `worker_threads` pool, replace `dns.lookup` with `dns.resolve` to take DNS off the pool entirely, or stream/chunk large compression.

## Senior Pitfalls

- Believing async `fs` is "OS async" and infinitely scalable — it competes for 4 threads.
- Putting `dns.lookup` (default for `http` requests via hostname) on the critical path under crypto load and blaming the network.
- Setting `UV_THREADPOOL_SIZE` at runtime and expecting it to apply.
- Conflating event-loop lag with pool saturation — they are independent failure modes needing different fixes.

## See Also
- [[libuv-and-event-loop]] — poll phase observes pool completions
- [[runtime-architecture]] — pool threads never touch V8
- [[process-lifecycle]] — startup is when the pool size is read
- [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]] — bounded worker pool theory
- [[03-computer-systems/operating-systems/io-models|I/O Models]] — why fs can't ride readiness
- [[03-computer-systems/networking/dns|DNS]] — lookup vs resolve resolver paths
