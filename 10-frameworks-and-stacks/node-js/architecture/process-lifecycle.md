# Process Lifecycle

A Node process has a definite arc: bootstrap the runtime, run the entry module, spin the event loop until it has nothing left to keep it alive, then tear down and exit with a code. The `process` object is your handle on every stage of that arc — signals, exit codes, fatal-error hooks, and the surprisingly subtle question of *what keeps the loop running*. Mastering the lifecycle is what separates a service that drains cleanly on deploy from one that drops in-flight requests.

## Bootstrap

Before any user code, the binary runs an internal bootstrap (largely the JS in `lib/internal/bootstrap/*`, much of it baked into a **V8 startup snapshot** for fast cold start). It wires up `process`, `globalThis`, the module loaders (CJS + ESM), the `Buffer`/`TextEncoder` globals, and installs the libuv loop. `UV_THREADPOOL_SIZE` is read here (see [[thread-pool]]). Only then is the **entry module** loaded.

The entry point is exposed as `process.argv[1]` and, for CommonJS, `require.main === module` is the idiom for "am I the script that was run directly?" (ESM uses `import.meta.main` in Node 22, or compares `import.meta.url`). Top-level code runs synchronously to completion; *then* the loop begins its first iteration — which is why top-level `setTimeout(0)` vs `setImmediate` ordering is a race (the loop hasn't started yet).

## The process Object

A global `EventEmitter` exposing the OS-process boundary:

- **Identity/env**: `process.pid`, `process.argv`, `process.env`, `process.cwd()`, `process.platform`, `process.execPath`.
- **Streams**: `process.stdin/stdout/stderr` — note these are *not* uniformly async; a pipe stdout is async, a file is sync, a TTY may be sync. This asymmetry causes "logs lost on crash" bugs.
- **Scheduling/diagnostics**: `process.nextTick`, `process.hrtime.bigint()`, `process.memoryUsage()`, `process.resourceUsage()`.
- **Lifecycle events**: `beforeExit`, `exit`, `uncaughtException`, `unhandledRejection`, `warning`, plus signal events.

## Signals

POSIX signals surface as events on `process`. libuv installs handlers and delivers them on the loop thread, so your handler runs as ordinary JS between phases (not in async-signal-unsafe context):

- **`SIGINT`** (Ctrl-C), **`SIGTERM`** (orchestrators' default stop) — the two you almost always handle for graceful shutdown.
- **`SIGHUP`**, **`SIGUSR1`** (Node reserves this to start the inspector), **`SIGUSR2`** (common app-defined trigger, used by nodemon).
- **`SIGKILL`** and **`SIGSTOP`** cannot be caught — no graceful shutdown is possible against `kill -9`.

A registered signal handler is itself an active handle that **keeps the loop alive**, which is sometimes surprising.

## Graceful Shutdown

The canonical pattern: stop accepting new work, drain in-flight work, release resources, let the loop empty naturally:

```js
const server = app.listen(3000);
function shutdown(signal) {
  console.log(`${signal} received, draining`);
  server.close(() => {           // stop accepting; fires when connections drain
    pool.end().then(() => process.exit(0));
  });
  setTimeout(() => process.exit(1), 10_000).unref(); // hard cap, don't hold loop open
}
process.on('SIGTERM', () => shutdown('SIGTERM'));
process.on('SIGINT',  () => shutdown('SIGINT'));
```

Key moves: `server.close()` rejects new connections but waits for active ones; the watchdog timer is `unref()`'d so it cannot *itself* keep the process alive; you avoid `process.exit()` mid-drain because it is abrupt.

## Exit Codes and Exit Paths

The loop ends when no **ref'd** handles/requests remain; Node then emits `beforeExit` (where you may schedule async work and *resurrect* the loop — it won't fire if you called `process.exit()`), then `exit` (synchronous only — async work here is ignored), then the process exits with the code. `process.exitCode = n` sets the code for natural exit; `process.exit(n)` exits *immediately and forcibly*, abandoning queued I/O (the classic cause of truncated stdout/log loss). Conventions: `0` success, `1` uncaught fatal, `>128` = `128 + signal number` (e.g. `130` from SIGINT).

## uncaughtException and unhandledRejection

- **`uncaughtException`**: a synchronous throw escaped all `try/catch` and reached the top. The handler lets you *log and exit*, not *recover* — after it the process is in an undefined state (a half-finished operation may hold corrupt invariants). Best practice: log, attempt fast cleanup, then exit non-zero and let your supervisor restart. Staying alive "to keep serving" is a leading cause of zombie processes with leaked state.
- **`unhandledRejection`**: a rejected Promise with no `.catch`/`reject` handler attached by the time it's checked. In Node 22 the **default is to throw it as an uncaughtException and crash** (`--unhandled-rejections=throw`), the right default — a swallowed rejection is a silent bug. Treat the handler as a logging hook, not a recovery mechanism.

## Keeping the Loop Alive: ref / unref

The loop stays alive while at least one **ref'd** handle exists. Each timer, socket, server, and child process is a handle. `handle.unref()` tells libuv "this handle should *not* by itself keep the process running"; `ref()` reverses it. Use `unref()` for background tasks that should never block shutdown — a metrics-flush interval, a keep-alive ping, the shutdown watchdog above. The mental model: **the process lives exactly as long as there is ref'd work to do.** An idle server stays up because its listening socket is a ref'd handle; close it and, with nothing else ref'd, the process exits on its own.

## Senior Pitfalls

- `process.exit()` before stdout/log streams flush → lost output; prefer setting `exitCode` and letting the loop drain.
- Trying to *recover* from `uncaughtException`/`unhandledRejection` instead of crashing cleanly.
- A stray ref'd `setInterval` (e.g. a metrics timer) holding the process open forever — forgetting to `unref()` it.
- Doing async work in the `exit` handler (silently ignored — `exit` is sync-only).

## See Also
- [[libuv-and-event-loop]] — handles, requests, and loop exit
- [[thread-pool]] — pool size fixed during bootstrap
- [[timers-and-microtasks]] — nextTick at startup; unref'd timers
- [[runtime-architecture]] — the process object as OS boundary
- [[03-computer-systems/operating-systems/interrupts-and-signals|Interrupts and Signals]] — POSIX signal delivery
- [[03-computer-systems/operating-systems/processes|Processes]] — process model and exit codes
