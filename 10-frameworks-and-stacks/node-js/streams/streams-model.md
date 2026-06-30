# Streams Model

Streams are Node's abstraction for processing data *incrementally* rather than all-at-once — the answer to "how do I handle a 50 GB file or an infinite socket on a process with 512 MB of heap?" The core idea is **bounded memory over unbounded data**: you move fixed-size chunks through a pipeline, never materializing the whole payload. Everything else — the four types, the two modes, backpressure — follows from that constraint.

## The Four Stream Types

- **Readable** — a source you pull from (`fs.createReadStream`, an HTTP request, `process.stdin`). Emits `'data'` chunks and a final `'end'`.
- **Writable** — a sink you push to (`fs.createWriteStream`, an HTTP response, `process.stdout`). You call `write()` and `end()`.
- **Duplex** — independent readable *and* writable sides over one resource (a TCP socket: you read what the peer sent, write what you send). The two sides have separate buffers.
- **Transform** — a Duplex where the writable input is *transformed* into the readable output (gzip, a cipher, a CSV parser). `zlib.createGzip()` is the canonical example.

Mental model: Readable and Writable are the primitives; Duplex is "both, unrelated"; Transform is "both, where output is a function of input."

## Flowing vs Paused Mode

A Readable is a state machine with two read modes:

- **Paused** (default): data sits in the internal buffer until you explicitly `read()`. You are in control of pace.
- **Flowing**: chunks are pushed at you as fast as they arrive, via `'data'` events.

Transitions are *implicit and a classic trap*: attaching a `'data'` listener, calling `.resume()`, or `.pipe()`-ing switches a stream to flowing. Calling `.pause()` or `.unpipe()` (with no `'data'` listener) returns it to paused. The danger: attach a `'data'` handler "just to log," and you silently start draining the stream — if there's no real consumer, data is read and discarded. **Async iteration** (`for await (const chunk of readable)`) is the modern, safer interface: it pulls in paused mode, respects backpressure automatically, and integrates with `try/catch`/`finally` for cleanup.

## Object Mode

By default streams carry `Buffer`/string chunks and `highWaterMark` counts *bytes*. In `objectMode: true`, each chunk is an arbitrary JS value and the watermark counts *objects*. This turns streams into a general lazy-sequence pipeline (parse → filter → map → write), the basis of libraries like the database cursor streams and many ETL flows. The cost: each object is a heap allocation, so object-mode watermarks are small (default 16) — they bound *count*, not memory, so a stream of huge objects can still blow the heap.

## The Event Model

```text
Readable:  'data' (flowing) · 'readable' (paused) · 'end' · 'error' · 'close'
Writable:  'drain' · 'finish' · 'error' · 'close' · 'pipe'/'unpipe'
```

Key invariants: `'end'` fires after all data is consumed; `'finish'` after `end()` is called and all writes flush; `'close'` after underlying resources are released. `'error'` does **not** auto-close or auto-propagate across `pipe()` — this is the root of most stream resource leaks.

## `pipe` vs `pipeline`

`src.pipe(dst)` wires a readable to a writable and propagates *data and backpressure* — but **not errors**, and it does not clean up the other streams on failure. A failed `pipe` chain commonly leaves dangling file descriptors. `stream.pipeline()` (and its promise form `require('node:stream/promises').pipeline`) is the correct modern API:

```js
import { pipeline } from 'node:stream/promises';
import { createReadStream, createWriteStream } from 'node:fs';
import { createGzip } from 'node:zlib';

await pipeline(
  createReadStream('big.log'),
  createGzip(),
  createWriteStream('big.log.gz')
); // errors reject; all streams destroyed on failure
```

`pipeline` forwards errors, destroys every stream on failure, and resolves/rejects once. Rule of thumb: never use bare `.pipe()` in production unless you are also manually wiring error handling on every stream.

## Why Streams Exist (Philosophy)

Streams trade *peak memory* and *latency-to-first-byte* for a more complex programming model. Reading a file with `fs.readFile` is simpler but O(filesize) memory and you wait for the whole read; a stream is O(highWaterMark) memory and starts producing output immediately. For request bodies, log processing, proxies, and media, streaming is not an optimization — it's the only model that doesn't fall over under load. The hard part, and the reason most stream bugs exist, is **backpressure**: keeping a fast producer from outrunning a slow consumer. That mechanism is its own topic.

## Senior Pitfalls

- Attaching `'data'` to "peek" and accidentally draining the stream.
- Using `.pipe()` and never handling `'error'` → leaked FDs.
- Forgetting object-mode watermarks count objects, not bytes.
- Mixing `await fs.readFile` patterns into streaming code, reintroducing unbounded memory.

## See Also

- [[backpressure]]
- [[file-system-and-networking]]
- [[buffers]]
- [[03-computer-systems/operating-systems/io-models|I/O Models]]
- [[03-computer-systems/concurrency-and-parallelism/event-loops|Event Loops]]
