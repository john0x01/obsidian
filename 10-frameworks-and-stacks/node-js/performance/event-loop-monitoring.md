# Event Loop Monitoring

Node serves all JavaScript on a single thread, so the event loop's responsiveness *is* the application's latency budget. Every callback, promise continuation, and timer waits its turn behind whatever is currently running. The two numbers that quantify this — **event loop delay** (lag) and **event loop utilization** (ELU) — are the most important health signals a Node service emits, more diagnostic than CPU% because a process can be "100% busy doing nothing useful for new work" while CPU looks calm.

## What the loop actually does, and where lag comes from

libuv runs the loop in ordered phases (timers → pending callbacks → poll → check → close), and between each callback Node drains the **microtask queue** (resolved promises, `queueMicrotask`) and the **`process.nextTick`** queue (which drains *before* microtasks). **Lag** is the gap between when a callback was *due* to run and when it *actually* ran — time stolen by whatever held the thread. The causes:

- **Synchronous CPU work** — a JSON.parse of a huge payload, a sync crypto/zlib call, a tight loop, a regex (see ReDoS). The loop cannot advance until it returns.
- **GC pauses** — a major mark-sweep stop-the-world pause freezes the thread; under memory pressure these stack up and *look* like random lag spikes (correlate with `--trace-gc`).
- **Microtask / nextTick floods** — a recursive `process.nextTick` or a promise chain that keeps scheduling more microtasks starves the loop *entirely*, because Node fully drains those queues before returning to the next phase. This is "the loop is spinning but timers and I/O never get serviced."
- **Sync I/O** — `fs.readFileSync`, `child_process.execSync` block libuv's thread directly.

## Measuring lag: monitorEventLoopDelay

The precise instrument is `perf_hooks.monitorEventLoopDelay()`, which uses a high-resolution timer *inside libuv* and records into an HDR histogram — far more accurate than the naive `setTimeout` self-timing trick, and it captures the *distribution*, which is what matters for tail latency.

```js
const { monitorEventLoopDelay } = require('node:perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 }); // ms sampling
h.enable();
// ...later, scrape periodically...
console.log(h.mean / 1e6, h.percentile(99) / 1e6, h.max / 1e6); // ns → ms
h.reset(); // reset between scrape windows so percentiles aren't lifetime-cumulative
```

All values are in **nanoseconds**. Healthy idle lag is sub-millisecond; sustained p99 of tens of ms means handlers are blocking. Reset after each scrape or your percentiles become a useless all-time average.

## Measuring saturation: eventLoopUtilization

`performance.eventLoopUtilization()` (ELU) returns cumulative `{ active, idle, utilization }`. **Utilization is the fraction of time the loop was *not* idle (not parked in the poll phase waiting for I/O).** Diff two readings to get the rate over an interval:

```js
const { performance } = require('node:perf_hooks');
let last = performance.eventLoopUtilization();
setInterval(() => {
  const now = performance.eventLoopUtilization();
  const elu = performance.eventLoopUtilization(now, last); // delta
  last = now;
  metrics.gauge('event_loop.utilization', elu.utilization); // 0..1
}, 5000);
```

ELU near 1.0 means the thread is saturated with synchronous work and has no spare capacity — the single most reliable **autoscaling and load-shedding signal** for Node, because unlike CPU% it directly reflects "can this instance accept more work?". Lag tells you *you're already late*; ELU tells you *how close to the cliff you are*.

## The two signals together

| Signal | Answers | Reacts to |
| --- | --- | --- |
| Loop delay (lag) | "Are callbacks running late *right now*?" | the symptom — queuing already happening |
| ELU | "How saturated is the thread?" | the cause — leading indicator before lag spikes |

Use **lag for SLO/alerting** (it correlates with user-visible p99) and **ELU for capacity decisions** (scale out / shed load when it trends past ~0.7-0.8). Watching only one misleads: a service can have low average lag but ELU climbing toward saturation, with a latency cliff one traffic bump away.

## Keeping the loop responsive

- **Move CPU-bound work off-thread.** Offload heavy computation to a `worker_threads` pool, or to a child process / native addon, so the main loop keeps servicing I/O.
- **Yield long jobs.** Chunk a big loop and `await` a macrotask boundary (e.g. `setImmediate`) between chunks so I/O and timers interleave. Prefer `setImmediate` over `process.nextTick`/microtasks for yielding — nextTick and microtasks do *not* yield to the poll phase, so they can't relieve I/O starvation.
- **Never call the sync variants in request paths** (`*Sync` fs, sync crypto/zlib on large inputs, `JSON.parse`/`stringify` on multi-MB payloads). Stream or chunk instead.
- **Bound concurrency** so you don't queue thousands of pending continuations whose scheduling overhead becomes its own tax.

## Measuring in production

Sample `monitorEventLoopDelay` percentiles and ELU deltas on a fixed interval and export them as gauges/histograms to your metrics pipeline; alert on lag p99 and ELU thresholds. The histogram approach is production-safe (negligible overhead, no sampling of user code). Correlate lag spikes with `--trace-gc` output and request-rate metrics to distinguish GC pauses from genuine CPU overload from a nextTick flood — the remediation differs for each.

## See Also
- [[profiling-and-diagnostics]]
- [[memory-and-gc]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime & Event Loop]]
- [[07-performance-engineering/tail-latency|Tail Latency]]
- [[08-quality-and-operations/observability/metrics|Metrics]]
- [[01-programming-foundations/languages/javascript/concurrency/web-workers|Web Workers / Worker Threads]]
