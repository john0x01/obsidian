# Profiling and Diagnostics

Profiling Node is the discipline of turning a vague "it's slow" into a precise statement about *which function, on which thread, holding which lock or awaiting which I/O, is eating wall-clock time*. The hard part is rarely the tooling — it is having a methodology that distinguishes CPU-bound stalls from event-loop starvation from downstream latency, because each demands a different fix.

## The mental model: where can time go?

A Node request's latency decomposes into: time the JS thread spends *executing* (on-CPU), time it spends *blocked* waiting (off-CPU: I/O, locks, downstream services), and time it spends *queued* behind other work because the single event loop was busy. A profiler that only measures on-CPU time will happily show you a flat profile while your p99 is 4 seconds — because the bottleneck was queuing, not computation. Always first answer "is this CPU-bound or latency-bound?" before choosing a tool.

## --inspect and Chrome DevTools

`node --inspect` (or `--inspect-brk` to break before user code) opens the V8 Inspector protocol on a WebSocket. Connect via `chrome://inspect`, `edge://inspect`, or the VS Code debugger. From the DevTools Performance/Profiler tab you can record a **CPU profile** (a sampling profile, default 1 kHz) and a **heap snapshot**. The Inspector also drives programmatic capture via the `inspector` module's `Session`, which is how you take profiles in production without a UI:

```js
const { Session } = require('node:inspector/promises');
const session = new Session();
session.connect();
await session.post('Profiler.enable');
await session.post('Profiler.start');
// ...exercise the workload...
const { profile } = await session.post('Profiler.stop'); // .cpuprofile JSON
```

The `.cpuprofile` is the canonical artifact — save it and open it in DevTools or speedscope later.

## Flame graphs and reading a CPU profile

A **flame graph** plots stacks on the y-axis (caller below callee) and *aggregate sample count* on the x-axis. Width = time; ordering is alphabetical, not chronological, so a flame graph is a statistical summary, not a timeline. Look for **wide plateaus** (a single frame consuming many samples = a hotspot) and **wide towers that recur** (a hot call path). The inverse, an **icicle graph**, hangs downward. A key trap: in a sampling profiler, anything *blocked off-CPU* (an `await` on a slow query) produces *no samples* and is invisible — the flame graph looks idle precisely when you are latency-bound.

## --prof and the tick processor

`node --prof` writes a low-overhead V8 tick log (`isolate-*.log`) by sampling the instruction pointer and tagging each tick with its V8 state (JS, GC, compiler, C++). Post-process with `node --prof-process isolate-*.log`. The output's gold is the **bottom-up "ticks from this and its children"** section and the breakdown of JS vs C++ vs GC ticks. High GC ticks point you at allocation pressure; high "Unknown"/C++ ticks often mean time in native bindings or syscalls.

## clinic.js

The `clinic` toolkit wraps the primitives into opinionated diagnostics: **Doctor** samples event-loop delay, CPU, memory and heuristically classifies the problem ("event loop blocked", "GC pressure", "I/O bound"); **Flame** generates a flame graph from `--prof`-style data; **Bubbleprof** visualizes async operation latency by grouping async resources (built on async_hooks), which is the rare tool that surfaces *off-CPU* waiting. Reach for Doctor first to get a hypothesis, then Flame or Bubbleprof to confirm.

## perf_hooks, diagnostics_channel, trace events

- **`perf_hooks`** is the in-process User Timing / Performance Timeline API: `performance.mark`/`measure`, `PerformanceObserver`, plus Node-specific entries (`gc`, `http`, DNS) and the histogram/event-loop helpers covered in the event-loop note. Use it to emit *durable* metrics, not for one-off investigation.
- **`diagnostics_channel`** is a zero-overhead-when-unsubscribed pub/sub bus. Core modules and libraries (undici, Fastify) publish channels you can subscribe to for structured timing without monkey-patching. It is the modern replacement for hooking internals.
- **Trace events** (`node --trace-event-categories v8,node,node.async_hooks`) emit a Chrome `trace_events`-format file (`node_trace.*.log`) viewable in `chrome://tracing` or Perfetto — a *timeline* view (unlike flame graphs), excellent for seeing GC pauses interleaved with your handlers.

## A methodology for finding hotspots

1. **Reproduce under load.** A single request rarely reveals the hotspot; profile under a representative load test so sampling has enough ticks.
2. **Classify first.** Run clinic Doctor or check event-loop delay (see below). CPU-bound, GC-bound, and latency-bound each route to a different tool.
3. **Capture the cheapest artifact that answers the question.** `--prof` or an inspector `.cpuprofile` for CPU; trace events for pause timelines; Bubbleprof for async waiting.
4. **Read top-down to localize, bottom-up to attribute.** Find the wide frame, then ask whether the cost is *self* time (optimize the function) or *children* (optimize what it calls or how often).
5. **Fix one thing, re-measure.** Profiles drift; never stack speculative fixes.

Senior pitfalls: trusting a flame graph that is idle (you were off-CPU, profile latency instead); profiling in dev where the JIT and data shapes differ from prod; forgetting that the sampling profiler shares the JS thread, so a blocked loop under-samples itself; and optimizing a wide frame that is actually correct, cheap-per-call, but called a million times — the fix is *fewer calls*, not a faster function.

## See Also
- [[memory-and-gc]]
- [[event-loop-monitoring]]
- [[07-performance-engineering/profiling|Profiling (general)]]
- [[01-programming-foundations/languages/javascript/engine-internals/v8-architecture|V8 Architecture]]
- [[08-quality-and-operations/observability/debugging-production|Debugging Production]]
- [[07-performance-engineering/tail-latency|Tail Latency]]
