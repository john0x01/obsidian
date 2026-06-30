# Buffers

`Buffer` is Node's representation of raw binary data: a fixed-length region of memory holding bytes, sitting *outside* V8's normal object heap. It exists because JavaScript, designed for the browser, historically had no way to hold binary octets — strings are UTF-16, numbers are doubles — yet servers must speak bytes: TCP frames, file contents, crypto, image data. Buffer was Node's pre-TypedArray answer, and it has since been retrofitted onto the standard `Uint8Array`.

## The TypedArray / ArrayBuffer Relationship

The modern, accurate mental model:

```text
ArrayBuffer        — a raw block of memory (just bytes, no interpretation)
   ▲
Uint8Array         — a typed *view* over an ArrayBuffer (each element = 1 byte)
   ▲
Buffer             — a Uint8Array subclass + Node-specific methods
```

`Buffer extends Uint8Array`, so every Buffer *is* a `Uint8Array` and works anywhere one is accepted (WebCrypto, `fetch` bodies, `TextDecoder`). It adds Node conveniences: `toString(encoding)`, `readInt32BE`/`writeUInt16LE` and friends, `Buffer.concat`, and the encoding-aware constructors. `buf.buffer` exposes the underlying `ArrayBuffer`; `buf.byteOffset`/`buf.byteLength` locate the view within it — important because pooled buffers share one backing `ArrayBuffer` (see below), so `buf.buffer` is usually *larger* than `buf`.

## Why Buffer Lives Off-Heap

Buffer memory is allocated by Node's C++ layer, not V8's GC heap. This is deliberate: I/O (sockets, files via the thread pool, native addons) needs stable, pointer-addressable memory that the moving garbage collector won't relocate mid-operation, and it lets large binary payloads bypass V8 heap-size limits. A 1 GB buffer doesn't count against the `--max-old-space-size` budget. The GC still tracks the small JS wrapper object and frees the backing store when it's collected, but the bulk bytes live in native memory.

## Allocation and the Internal Pool

Three creation paths, with a critical difference:

- `Buffer.alloc(n)` — allocates `n` bytes and **zero-fills** them. Safe, slightly slower.
- `Buffer.allocUnsafe(n)` — allocates `n` bytes **without zeroing**. Faster, but the memory contains *whatever was there before* — potentially fragments of previously-freed buffers (old request bodies, decrypted secrets).
- `Buffer.from(...)` — from a string/array/ArrayBuffer/existing buffer (copies, except when wrapping an ArrayBuffer).

`allocUnsafe` for small sizes is served from an **internal pool**: Node pre-allocates an 8 KB slab (`Buffer.poolSize`) and hands out *slices* of it for allocations smaller than `poolSize >>> 1` (4 KB). This amortizes allocation cost across many small buffers. The consequence: small `allocUnsafe` buffers may share one backing `ArrayBuffer`, which is why `buf.byteOffset` matters and why you must never assume `buf.buffer` is exclusively yours.

## The `allocUnsafe` Security Note

Because `allocUnsafe` exposes uninitialized memory, returning or transmitting such a buffer before fully overwriting it can leak unrelated data — a real CVE class in Node's history (the old `new Buffer(number)` defaulted to unsafe, hence its deprecation). Rule: use `allocUnsafe` *only* when you will immediately and completely overwrite every byte (e.g. you're about to `read()` exactly `n` bytes into it). Otherwise use `alloc`. Never hand an `allocUnsafe` buffer to user-facing output without filling it.

## Encodings

A Buffer is bytes; an *encoding* is the interpretation applied when converting to/from strings:

- `utf8` (default), `utf16le`, `latin1`/`binary` (1:1 byte↔codepoint for 0–255), `ascii`.
- `hex`, `base64`, `base64url` — text representations of binary.

`buf.toString('utf8')` and `Buffer.from(str, 'utf8')` are the conversions. A subtle streaming bug: a multi-byte UTF-8 character can be split across two chunks, so `chunk.toString()` per-chunk corrupts it — use `string_decoder.StringDecoder`, which buffers partial code points across chunk boundaries.

## Zero-Copy Slices

`buf.subarray(start, end)` (and the older `buf.slice`) returns a **new view over the same memory** — no copy. Mutating the slice mutates the parent and vice-versa:

```js
const a = Buffer.from([1,2,3,4]);
const b = a.subarray(1, 3); // view of bytes [2,3], shares memory
b[0] = 99;                  // a is now [1,99,3,4]
```

This is fast (parsing a protocol header without copying the payload) but a sharp edge: aliasing bugs and accidental mutation. When you need an independent copy, `Buffer.from(buf)` or `Uint8Array.prototype.slice` (which *does* copy for TypedArrays). Note the historical footgun: `Buffer.prototype.slice` was a *view* (unlike `Array.slice`), which surprised everyone; `subarray` is the explicit, recommended name today.

## Senior Pitfalls

- Using the deprecated `new Buffer()` constructor.
- `chunk.toString()` per stream chunk splitting multi-byte chars.
- Assuming `buf.buffer` is the same length as `buf` (pooling).
- Returning unfilled `allocUnsafe` memory (info leak).
- Forgetting `subarray` aliases memory.

## See Also

- [[streams-model]]
- [[file-system-and-networking]]
- [[01-programming-foundations/languages/javascript/concurrency/shared-memory-and-atomics|Shared Memory and Atomics]]
- [[03-computer-systems/computer-architecture/memory-hierarchy|Memory Hierarchy]]
