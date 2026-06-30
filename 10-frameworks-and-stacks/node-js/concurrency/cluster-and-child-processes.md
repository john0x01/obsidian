# Cluster And Child Processes

`child_process` and `cluster` are Node's **process-based** parallelism tools: instead of threads inside one process, you spawn separate OS processes, each with its own PID, memory space, and V8 runtime. The defining trade-off versus Worker Threads is *isolation versus communication cost* — processes give you crash isolation, independent memory limits, and the ability to run non-Node programs, at the price of heavier spawning and IPC that always serializes.

## `child_process`: The Four Spawners

Everything reduces to `spawn`; the others are conveniences layered on top.

- **`spawn(cmd, args, opts)`** — the primitive. Launches a program and gives you `stdin`/`stdout`/`stderr` as *streams*. No shell by default (pass `shell: true` to invoke one). Streaming means it handles unbounded output without buffering the whole thing in memory.
- **`execFile(file, args, cb)`** — runs a binary directly, buffers all output into a callback. Convenient for short output; dangerous for large output (it enforces `maxBuffer`, default 1 MB, and errors past it).
- **`exec(cmd, cb)`** — like `execFile` but runs the string *through a shell* (`/bin/sh -c`). This is the one that invites shell-injection: never interpolate untrusted input into the command string. Prefer `execFile`/`spawn` with an args array, which never touches a shell.
- **`fork(modulePath, args, opts)`** — a specialization of `spawn` that launches a *new Node process* running a JS module and automatically wires up an **IPC channel**. This is the only one of the four where parent and child can `process.send()` / `'message'` structured-clone objects to each other.

`stdio` controls the child's three standard streams. `'pipe'` (default) exposes them as streams on the parent; `'inherit'` shares the parent's terminal directly (great for passing through a build tool's output); `'ignore'` discards. You can also pass an array to configure each fd independently and add an `'ipc'` slot. Communication with non-Node children is via these streams; with forked Node children it is via the IPC message channel, which can also pass server handles (a `net.Server` or socket) between processes — the mechanism `cluster` is built on.

## `cluster`: Sharing a Listening Socket

`cluster` exists for one job: run multiple Node processes that all serve the **same port**, so a single-threaded runtime can saturate a multi-core box. It is `fork()` under the hood, with the **primary** process orchestrating **workers**.

The clever part is socket sharing. You cannot have two processes independently `bind()` the same port — the second gets `EADDRINUSE`. So when a cluster worker calls `server.listen(port)`, the call is intercepted: the *primary* owns the actual listening socket, and workers receive connections from it.

```
                ┌──────────────────────┐
   :8080 ──────►│   Primary (owns fd)  │
                └──────────┬───────────┘
        distributes accepted connections
          ┌────────────┼────────────┐
       Worker 1     Worker 2     Worker 3   (each = full Node process)
```

Two distribution schemes exist. The default on most platforms is **round-robin** (`SCHED_RR`): the primary `accept()`s connections and hands each to a worker, which spreads load evenly and avoids the thundering-herd problem. The alternative (`SCHED_NONE`, the Windows default historically) passes the *listening socket fd* to all workers and lets the **OS kernel** decide which worker's `accept()` wins — simpler but prone to lopsided distribution because the kernel favors busy CPUs. Round-robin is preferred precisely because OS-level balancing tends to clump connections.

## Process vs Thread Parallelism

| | Child process / cluster | Worker Threads |
|---|---|---|
| Memory | Separate address space | Separate Isolate, same process |
| Crash blast radius | Isolated (one worker dies, others live) | A hard crash can take the process |
| Spawn cost | High (`fork`+`exec`, new runtime) | Lower (no new process) |
| Data sharing | Copy only (IPC serialize) | Copy *or* `SharedArrayBuffer` |
| Can run | Any program | Only JS |

The senior mental model: **`cluster` scales an HTTP server across cores; Worker Threads offload a CPU hot spot within one server.** They are not competitors — a common production shape is N cluster workers, each spawning a small Worker pool for hashing or rendering.

## Pitfalls and Misconceptions

- **`cluster` is not a substitute for a process manager.** It does not restart crashed workers for you, balance across machines, or do zero-downtime reloads unless you write that logic (listen for `'exit'` and re-`fork`). In containerized deployments people increasingly skip `cluster` entirely and run one process per container, letting the orchestrator (Kubernetes, a load balancer) do the spreading — simpler, more observable, and the scheduler already exists.
- **Shared in-memory state is a lie across workers.** Sessions, caches, rate-limit counters live in *one* worker's heap. Externalize to Redis or sticky-route. WebSocket/long-poll setups need sticky sessions or a pub/sub backplane because a client's connection lands on one specific worker.
- **`exec` with interpolated input is a shell-injection hole.** Reach for `execFile`/`spawn` + args array by default.
- **Zombie children.** A spawned child outlives a crashed parent unless you handle `'exit'`/signals; orphans accumulate.
- **IPC serializes.** `process.send` uses structured clone over a pipe — same copy cost as Worker messaging, plus a context switch. Chatty parent-child protocols are slow.

## See Also

- [[worker-threads]]
- [[async-context]]
- [[03-computer-systems/operating-systems/processes|Processes]]
- [[03-computer-systems/operating-systems/ipc|IPC]]
- [[03-computer-systems/networking/load-balancing|Load Balancing]]
- [[03-computer-systems/operating-systems/threads|Threads]]
