# V8 Architecture

V8 is the engine that turns JavaScript source into executing machine code and manages the objects it produces. It exists because shipping a fast, portable JS implementation means *not* writing an ahead-of-time compiler (the code arrives over the wire at page load) and *not* paying full optimizing-compilation cost for code that runs once. V8's whole shape is a response to that tension: start executing almost immediately, then spend optimization effort only where it pays off.

## The Pipeline

```
source text
  -> Scanner (tokens) -> Parser -> AST
  -> Ignition: AST -> bytecode  (and runs it, an interpreter)
        | collects type feedback into feedback vectors
        v
  -> Sparkplug: bytecode -> unoptimized machine code (baseline)
  -> Maglev / TurboFan: optimizing compilers (speculative, feedback-driven)
        ^ deoptimize back to Ignition when assumptions break
```

The key design choice: V8 is a **tiered, profile-guided JIT**. It does not compile everything optimized up front. It interprets bytecode first (fast to produce, low memory), watches which functions are *hot* and *what types they actually see*, then re-compiles those into specialized machine code. Cold code never pays optimization cost; hot code gets it. This is why micro-benchmarks lie: the first runs are interpreted, and the steady state is a different program.

- **Scanner / Parser → AST**: tokenizes and builds an Abstract Syntax Tree, with **lazy parsing** (pre-parsing) so function bodies that aren't called yet are only skimmed. (See [[parsing-and-bytecode]].)
- **Ignition**: a register-based bytecode interpreter. It generates compact bytecode and executes it, accumulating **type feedback** in *feedback vectors* attached to each function. Bytecode is the durable representation V8 keeps in memory — far smaller than machine code.
- **Sparkplug → Maglev → TurboFan**: the compiler tiers. Sparkplug is a non-optimizing baseline that emits machine code straight from bytecode for a quick speedup; Maglev is a mid-tier optimizer; TurboFan is the heavyweight optimizing compiler doing inlining, escape analysis, and speculative type specialization. (See [[jit-compilation]].)

## Isolates, Contexts, and the Heap

V8's runtime structure is layered, and the boundaries matter for embedders and for reasoning about memory:

- **Isolate** — an independent instance of the engine: its own heap, its own GC, and a strict *single-thread-at-a-time* discipline (one isolate is entered by one thread at a time). Two isolates share *nothing*. This is the unit of isolation behind a Web Worker, a Node Worker thread, and (with extra machinery) Cloudflare-style "isolate per request" models — far cheaper than an OS process.
- **Context** — a sandboxed global environment *within* an isolate: its own `globalThis`, built-ins, and globals. Multiple contexts can coexist in one isolate (e.g. multiple iframes / Node `vm` contexts) and share the heap but not their global scopes. V8 enforces a **security boundary** between contexts.
- **Heap** — per-isolate, generationally partitioned (new space, old space, plus large-object, code, and read-only spaces). The [[garbage-collector]] operates per isolate, which is why one worker's GC pause can't be hidden by another's idle time and why cross-isolate references aren't allowed — there is no shared object graph to trace.

**Handles and `v8::HandleScope`** are how embedder C++ holds heap references safely across a *moving* GC: a handle is an indirection the collector updates when it relocates an object. This is the seam between "JS objects" and native code.

## Snapshots

Building all the built-in objects (`Object`, `Array`, `Math`, …) at every startup is slow. V8 serializes a pre-initialized heap into a **startup snapshot** and deserializes it on boot — this is a major part of why Node and Chrome start fast. Embedders can also create **custom snapshots** (e.g. Node's user-land snapshot, or a framework baking its module graph in) to skip re-running initialization. (Detail in [[parsing-and-bytecode]].)

## Embedders

V8 is a *library*; it only does anything inside a host that supplies the event loop and platform APIs:

- **Chrome / Chromium** — pairs V8 with Blink (DOM, rendering, Web APIs) and the browser event loop. V8 knows nothing about the DOM; Blink exposes it.
- **Node.js** — pairs V8 with libuv (the event loop, thread pool, async I/O) and C++ bindings for the file system, network, etc. Same engine, entirely different host surface (no DOM, has `Buffer`/streams/`process`).
- **Deno / Bun (Bun uses JavaScriptCore, not V8)** — Deno embeds V8 via Rust (`rusty_v8`) with a different std library and security model. Worth knowing that "JS engine" ≠ "runtime": the language semantics come from V8, the *capabilities* from the embedder.

**Senior implications**: the same code is fast in one runtime and slow in another only because of host APIs and GC scheduling, not the language. Reaching for Workers buys true parallelism precisely because each gets its own isolate/heap — with no shared mutable objects, only message passing or `SharedArrayBuffer`. And "it's slow on first call, fast later" is the tiering pipeline doing exactly what it was designed to do.

## See Also
- [[parsing-and-bytecode]] — source to AST to Ignition bytecode
- [[jit-compilation]] — the optimizing compiler tiers
- [[hidden-classes-and-inline-caches]] — how feedback specializes object access
- [[garbage-collector]] — the per-isolate heap manager
- [[runtime]] — engine vs host runtime split
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]] — why V8 chose JIT
