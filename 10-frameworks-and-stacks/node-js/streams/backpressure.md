# Backpressure

Backpressure is the mechanism by which a slow consumer tells a fast producer to *slow down*, so that data in flight never exceeds a bounded buffer. It is the single most important — and most ignored — property of Node streams: get it wrong and a streaming pipeline that "works" in tests silently accumulates unbounded memory in production and OOM-kills the process under load.

## The Internal Buffer and `highWaterMark`

Every Readable and Writable owns an internal buffer with a threshold called `highWaterMark` (HWM). For byte streams the HWM is in bytes (default 64 KB for `fs`/socket streams, 16 KB historically for generic streams); in object mode it's a count (default 16). The HWM is **not a hard cap** — it's a *signal level*. The stream keeps accepting data past it; what changes is the advisory it returns to you. The contract is cooperative: the buffer can exceed HWM if you ignore the signal, which is exactly how runaway memory happens.

## `write()` Return Value and `'drain'`

The Writable contract lives in two values:

```js
const ok = writable.write(chunk);
// ok === true  → buffer below HWM, keep writing
// ok === false → buffer at/over HWM, STOP and wait for 'drain'
```

When `write()` returns `false`, you are obligated to stop calling `write()` until the `'drain'` event fires, signaling the buffer has emptied below HWM. Ignoring `false` is *legal* — Node still buffers the chunk — but each ignored write grows the internal buffer without limit. A loop that writes a million chunks while ignoring the return value buffers all million in memory:

```js
// WRONG — ignores backpressure, buffers everything
for (const row of millionRows) ws.write(serialize(row));

// RIGHT — manual backpressure
for (const row of millionRows) {
  if (!ws.write(serialize(row))) {
    await once(ws, 'drain'); // pause until consumer catches up
  }
}
```

On the Readable side, the symmetric signal is `read()` returning `null` (buffer empty) and the `'readable'`/`'end'` events; in flowing mode, `.pause()`/`.resume()` are how a consumer applies and releases pressure.

## How `pipe`/`pipeline` Propagate Backpressure

This is the payoff of the abstraction. `src.pipe(dst)` automatically does the `write()`/`'drain'` dance for you: when `dst.write()` returns `false`, `pipe` calls `src.pause()`; when `dst` emits `'drain'`, it calls `src.resume()`. The flow self-regulates to the speed of the slowest stage:

```text
ReadStream ──chunk──► Gzip ──chunk──► WriteStream
   (fast)              (cpu)            (slow disk)
     ▲                                     │
     └────── pause/resume backpressure ────┘
        slowest stage sets the whole rate
```

`stream.pipeline()` does the same propagation *and* adds error forwarding and guaranteed destruction. `for await...of` on a Readable also respects backpressure: the loop body's `await` naturally throttles pulling. The reason to prefer `pipeline`/async-iteration over hand-rolled `write()` loops is precisely that they make correct backpressure the default rather than something you must remember.

## What Goes Wrong Without It

The failure mode is **unbounded buffering**, and it's insidious because it depends on the *relative speeds* of producer and consumer:

- Producer faster than consumer + ignored backpressure → internal buffer grows without bound → heap climbs → GC thrash → OOM kill (often only under production load, never in dev where data is small).
- A common real case: piping an HTTP download to a slow disk, or `res.write()`-ing a large dynamically-generated payload in a loop. Memory tracks the *gap* between produce and consume rates integrated over time.
- Crossing into non-stream APIs (collecting all chunks into an array, `JSON.stringify` of a huge accumulated buffer) discards backpressure entirely — you've reintroduced the all-at-once memory model the stream was meant to avoid.

## Tuning and Philosophy

HWM is a latency/throughput/memory knob. Larger HWM → fewer pause/resume cycles and higher throughput, at the cost of more resident memory and more latency before backpressure engages. Smaller HWM → tighter memory and faster reaction, but more syscalls/event churn. Defaults (64 KB) are well-chosen for typical file/socket workloads; only tune with a measured reason. The philosophy is that Node makes backpressure *cooperative, not enforced*: the runtime gives you the signal and trusts you (or `pipeline`) to honor it. That trust is also the footgun — the API will happily let you buffer gigabytes if you ignore the return value.

## Senior Pitfalls

- Treating `write()` as fire-and-forget; ignoring its boolean.
- Awaiting `'drain'` when `write()` returned `true` — you'll hang, since `'drain'` only fires after a `false`.
- Assuming HWM is a hard memory cap (it's a signal threshold).
- Hand-rolling `.pipe()` loops instead of `pipeline`, losing both error handling and clean teardown.

## See Also

- [[streams-model]]
- [[file-system-and-networking]]
- [[buffers]]
- [[03-computer-systems/operating-systems/io-models|I/O Models]]
- [[03-computer-systems/networking/tcp|TCP]]
