# Parsing and Bytecode

Before V8 can run a line of JavaScript it must turn text into something executable, and *when* it does that work dominates page-load and Node-startup time. Parsing is expensive — historically a large fraction of cold start — so V8's front end is engineered around one principle: **do the least parsing work that lets the program start, and defer the rest until proven necessary.**

## Scanner and the Two-Speed Parser

The **scanner** reads the raw character stream (handling UTF-16, comments, line terminators) and emits **tokens**. The parser consumes tokens and builds an **AST** (Abstract Syntax Tree) — a tree of nodes (`FunctionDeclaration`, `BinaryExpression`, …) that captures structure and scope.

The crucial optimization is **lazy (deferred) parsing** backed by the **pre-parser**:

- **Pre-parsing** does a fast, shallow pass over a function body: enough to find its boundaries, validate syntax, and record which outer-scope variables it references (so closures resolve correctly) — but it does *not* build the full AST or allocate the inner data structures.
- **Full (eager) parsing** builds the complete AST and is only done when a function is actually about to be compiled/run.

```
function load() { ... }   <- eagerly parsed (it runs at startup)
function rarelyUsed() {   <- pre-parsed only: skimmed, boundaries noted
  // body's full AST built lazily, the first time it's called
}
```

So a function that's declared but never called costs you a skim, not a full parse. The risk: a function parsed lazily that *does* run gets parsed **twice** (pre-parse + full parse) — a small re-parse tax. V8 heuristically *eagerly* parses functions it predicts will run immediately; the classic hand-optimization is wrapping a startup-critical function in parentheses (an IIFE-like hint, the basis of tools' "optimize.js") so V8 eager-parses it. Top-level code is always eagerly parsed because it runs straight away.

## Why an AST, Then Bytecode

V8 does not interpret the AST directly, and it does not compile straight to machine code. **Ignition** lowers the AST to **bytecode** — a compact, linear instruction stream — and an interpreter executes that. This indirection is a deliberate space/time trade:

- **Machine code is large.** Compiling every function to native code up front bloats memory; on a page with thousands of rarely-run functions that's wasteful. Bytecode is *much* smaller (often several times smaller than equivalent machine code), so V8 keeps bytecode resident and only promotes hot functions to machine code.
- **Bytecode is the canonical form.** All later tiers (Sparkplug, Maglev, TurboFan) compile *from bytecode*, not from the AST, and deoptimization lands *back* on the bytecode interpreter. Bytecode is the stable pivot of the whole pipeline.

This is why V8's interpreter exists at all: cheap to produce, cheap in memory, "fast enough" until profiling proves a function deserves more.

## Ignition's Register Machine

Ignition is a **register-based** bytecode interpreter, not a stack machine. Operands live in a flat array of virtual **registers** (a register file per function activation, plus the **accumulator**, an implicit register that most instructions read/write to keep encodings short). Register machines typically need fewer instructions than stack machines for the same work (no constant push/pop shuffling), which means fewer dispatches.

```
// for: a + b
Ldar a        // load 'a' into the accumulator
Add  b         // accumulator = accumulator + b  (feedback slot attached)
Star result    // store accumulator into 'result'
```

Two structural details that matter for performance reasoning:

- **Feedback slots.** Operations like property loads, calls, and binary ops carry an index into a **feedback vector** — the per-function side table where Ignition records *what types it actually saw*. This is the raw material the optimizing tiers consume (and the substrate of [[hidden-classes-and-inline-caches]]). The interpreter is thus also the profiler.
- **Dispatch.** The interpreter is a tight loop reading bytecode and jumping to a handler per opcode; V8 generates these handlers themselves with its compiler backend so the interpreter is itself fast machine code.

## Code Cache and Snapshots

Two mechanisms avoid redoing front-end work across runs:

- **Code caching ("compilation cache").** Browsers cache the *bytecode* of scripts (not just the source) keyed to the resource, with cold/warm/hot tiers. On a repeat visit V8 can deserialize bytecode instead of re-scanning and re-parsing the source. The bytecode-cache layer is exactly why parsing-heavy sites benefit so much from a warm cache, and why `V8.compile` / `vm.Script` `cachedData` exists in Node.
- **Snapshots.** The built-in objects and the engine's initial heap are serialized into a **startup snapshot** and deserialized on boot rather than constructed by running JS each time. Embedders can build **custom snapshots** that bake in their own initialized state (Node's user snapshot; frameworks pre-loading a module graph) to cut startup further.

**Senior takeaways**: bundle size hurts twice — bytes to download *and* bytes to parse — so dead-code elimination and lazy chunking cut real parse cost, not just transfer. Shipping a giant module that's mostly unused still costs pre-parse time at top level if it's eagerly evaluated. And first-load vs repeat-load performance gaps are often the code cache, not the network.

## See Also
- [[v8-architecture]] — where parsing sits in the pipeline
- [[jit-compilation]] — tiers that compile from this bytecode
- [[hidden-classes-and-inline-caches]] — feedback vectors filled here, used there
- [[deoptimization]] — bailing back to the bytecode interpreter
- [[07-performance-engineering/frontend-performance|Frontend Performance]] — parse cost of large bundles
