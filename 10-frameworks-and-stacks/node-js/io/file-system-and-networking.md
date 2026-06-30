# File System And Networking

`fs` and `net`/`http` are Node's two big I/O surfaces, and they expose a foundational asymmetry: **disk I/O is faked-async via a thread pool, while network I/O is genuinely async via the OS event mechanism.** Understanding which calls block, which delegate to threads, and which ride the kernel's readiness notifications is the difference between a server that scales and one that stalls under a single slow filesystem.

## fs: Three API Flavors

The same operations come in three shapes:

- **Sync** (`fs.readFileSync`): blocks the entire event loop until done. Acceptable at startup (loading config) or in CLI scripts; catastrophic in a request handler, where one slow read freezes *all* concurrent connections.
- **Callback** (`fs.readFile(path, cb)`): the original async form, `cb(err, data)`.
- **Promises** (`fs.promises.readFile` / `import { readFile } from 'node:fs/promises'`): the modern default, `async/await`-friendly.

All three sit on the same C layer (`libuv`). For a senior, the rule is: promises everywhere in app code; sync only when blocking is provably fine; callbacks only when interfacing with old APIs.

## File Descriptors

Under the convenience methods, the OS model is file descriptors — small integers naming open kernel file objects. `fs.open` yields an `fd` (or a `FileHandle` in the promises API); `read`/`write`/`close` operate on it. `fs.readFile` is sugar that opens, reads to EOF, and closes for you. The pitfall: **descriptors leak**. Every `open` without a matching `close` consumes a process-limited resource (`ulimit -n`); under load this surfaces as `EMFILE: too many open files`. The promises `FileHandle` mitigates this in Node 20+/22 by registering with a `FinalizationRegistry` so the GC can close orphaned handles — but relying on GC for resource cleanup is a smell; close explicitly (or use streams/`pipeline`, which destroy on completion).

## The Thread-Pool Backing for Disk I/O

This is the crucial mechanism. There is no portable, truly-async syscall for *regular file* I/O the way there is for sockets, so `libuv` runs filesystem operations on a **thread pool** (default 4 threads, `UV_THREADPOOL_SIZE`, max 1024). When you `await readFile`, the call is dispatched to a pool thread that makes the blocking syscall; when it returns, the result is posted back to the event loop. Implications:

- Concurrent fs operations are limited by pool size: 5 simultaneous big reads with 4 threads means one waits.
- The pool is *shared* with DNS (`dns.lookup`), `crypto.pbkdf2`/`scrypt`, and `zlib`. A burst of `pbkdf2` can starve your file reads, and vice-versa — a non-obvious cross-coupling.
- CPU-bound work wrongly placed here saturates threads. Size `UV_THREADPOOL_SIZE` to your workload if you do heavy crypto/compression.

```text
JS: await readFile() ─► event loop ─► libuv ─► [thread pool: blocking read()] ─► back to loop ─► resolve
```

## net / http: True OS-Level Async

Sockets are different: the kernel offers readiness notification (`epoll` on Linux, `kqueue` on macOS, IOCP on Windows), so `libuv` registers the socket and is told when it's readable/writable — **no thread per connection**. This is the heart of Node's "one thread, thousands of connections" model: the event loop polls the kernel for ready sockets and runs your callbacks. `net.Server`/`net.Socket` are the TCP primitives; sockets are `Duplex` streams, so reading/writing them is the stream model with full backpressure.

`http` builds on `net`. The `http.Server` parses requests and gives you `(req, res)` where `req` is a Readable (the body) and `res` is a Writable — streaming bodies in and out without buffering whole payloads. On the client side, `http.request`/`fetch` send requests.

## Agent and Keep-Alive

TCP connection setup (handshake, and TLS adds more round-trips) is expensive. The `http.Agent` **pools and reuses** connections via HTTP keep-alive so successive requests to the same host skip reconnection. In Node 22, keep-alive is the default for the global agent. Key knobs: `keepAlive`, `maxSockets` (cap concurrent connections per host — too low throttles throughput, too high can exhaust the peer or local ports), and `maxFreeSockets`. The classic bug: creating a fresh `Agent` per request (or `http.request` with `agent: false`) defeats pooling and floods the host with new connections, often exhausting ephemeral ports. One shared agent, sized to the workload, is the pattern.

## Blocking vs Non-Blocking — The Through-Line

The unifying philosophy: Node keeps the *one JS thread* free at all costs. Non-blocking means "register interest, get called back when ready," and Node achieves it two ways — natively for sockets (kernel readiness) and *simulated* for files (thread pool). Anything that blocks the JS thread (`*Sync` calls, tight CPU loops) stalls every connection, because there's no other thread to take over. That is why "don't block the event loop" is Node's prime directive, and why knowing *which* I/O is thread-pool-backed vs kernel-async tells you exactly where your scaling ceiling sits.

## Senior Pitfalls

- `*Sync` fs calls inside request handlers.
- Leaking file descriptors → `EMFILE`.
- Thread-pool starvation between fs/crypto/zlib/dns.
- New `Agent` per request → port exhaustion, no connection reuse.
- Treating `dns.lookup` as free network I/O — it's on the *thread pool*, unlike `dns.resolve` which uses the network directly.

## See Also

- [[streams-model]]
- [[backpressure]]
- [[buffers]]
- [[03-computer-systems/operating-systems/io-models|I/O Models]]
- [[03-computer-systems/networking/sockets|Sockets]]
- [[03-computer-systems/concurrency-and-parallelism/thread-pools|Thread Pools]]
