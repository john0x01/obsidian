# NIO, Channels, And Selectors

`java.nio` ("New I/O," Java 1.4) is the buffer-and-channel API that lets a single thread manage thousands of connections through **non-blocking multiplexing** — the foundation under every high-performance Java network server. Where `java.io` (see [[io-streams-and-readers]]) gives you a blocking stream per connection, NIO gives you buffers, channels, and a selector that maps directly onto the OS's readiness-notification syscalls.

## Buffers: Direct Versus Heap

The unit of transfer is a `ByteBuffer`, a mutable cursor over a fixed-size region governed by four invariants: `0 ≤ mark ≤ position ≤ limit ≤ capacity`. The state machine trips people up: you `put` data (position advances), `flip()` (limit ← position, position ← 0) to switch to draining, `get` it, then `clear()` or `compact()` to refill. Forgetting `flip()` is the classic NIO bug.

Buffers come in two kinds:

- **Heap buffers** are backed by a Java `byte[]` on the GC heap. Cheap to allocate, but a syscall must first copy their contents to a temporary native buffer, because the GC can move the array.
- **Direct buffers** (`allocateDirect`) live in off-heap memory the OS can hand straight to a syscall — no copy. They cost far more to allocate and are freed only when a `Cleaner` reclaims the wrapper, so you pool them. Use direct buffers for long-lived, high-throughput channels; heap buffers for everything else.

## Channels And Zero-Copy

A `Channel` is a bidirectional conduit to a file or socket — `FileChannel`, `SocketChannel`, `ServerSocketChannel`, `DatagramChannel` — that reads into and writes from buffers. Channels also enable **zero-copy** bulk transfer: `FileChannel.transferTo` can move bytes from a file straight to a socket inside the kernel (via `sendfile`), never crossing into JVM memory — the trick behind fast static-file serving.

## Selectors And The Reactor

The heart of NIO is the `Selector`. A `SelectableChannel` in non-blocking mode registers with a selector for interest ops — `OP_READ`, `OP_WRITE`, `OP_ACCEPT`, `OP_CONNECT` — and one thread calls `select()`, which blocks until *any* registered channel is ready, returning the ready `SelectionKey` set. This is the **reactor pattern**, and it compiles down to the OS multiplexers described in [[03-computer-systems/operating-systems/io-models|I/O Models]]: `epoll` on Linux, `kqueue` on BSD/macOS, and increasingly `io_uring`.

```
        register(OP_READ)      +-----------+
 chan A ---------------------> |           | select() --> ready keys
 chan B ---------------------> | Selector  |              (A readable,
 chan C ---------------------> |           |               C writable)
        one thread   <-------- +-----------+
```

One thread, many connections — this is why NIO scales where thread-per-connection does not, and it is precisely the architecture that frameworks like Netty package into an event loop of selector threads. That is the same design as Node's single-threaded [[10-frameworks-and-stacks/node-js/architecture/libuv-and-event-loop|Node Event Loop]] and its `libuv` backend; the difference is that Java exposes the raw primitives and lets you choose the threading model, whereas Node bakes in exactly one. The broader contrast with blocking sockets is in [[03-computer-systems/networking/sockets|Sockets]] and [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]].

## Memory-Mapped Files

`FileChannel.map()` returns a `MappedByteBuffer` that maps a file region into the process's virtual address space, so reads and writes become ordinary memory accesses serviced lazily by the OS page cache — no explicit `read`/`write` syscalls. This is how databases and message logs (e.g. Kafka) get high-throughput persistence: the kernel handles paging and write-back. It leans entirely on the virtual-memory machinery, and inherits its hazards — a page fault on a network filesystem or a truncated file can raise errors mid-access.

## The Loom Reframing

NIO's callback/selector style is powerful but hard to write and read. Virtual threads (Project Loom) change the calculus: a virtual thread doing a *blocking* `SocketChannel.read()` unmounts from its carrier, so you can write straight-line blocking code that scales like a reactor (see [[io-streams-and-readers]]). Selectors remain essential inside frameworks and for the lowest-latency tier, but application code increasingly no longer needs to hand-write reactors — the JVM now offers both the manual primitive and the automatic scheduler.

## See Also
- [[io-streams-and-readers]]
- [[03-computer-systems/operating-systems/io-models|I/O Models]]
- [[10-frameworks-and-stacks/node-js/architecture/libuv-and-event-loop|Node Event Loop]]
- [[03-computer-systems/networking/sockets|Sockets]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
