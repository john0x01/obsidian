# JavaScript

Deep notes on the JavaScript language and its runtime — aimed at mastery: how the engine executes code, the language's semantics, modules, asynchrony, and parallelism. Not a syntax reference.

[← Back to Programming Foundations](../../README.md)

## Execution Model
- [Runtime & Event Loop](execution-model/runtime.md) — engine vs host, call stack, execution contexts, task/microtask queues
- [Garbage Collector](execution-model/garbage-collector.md) — V8 generational GC, the Scavenger, mark-compact, Orinoco, leaks

## Engine Internals (V8)
- [V8 Architecture](engine-internals/v8-architecture.md) — the parse → interpret → optimize pipeline, isolates, contexts
- [Parsing & Bytecode](engine-internals/parsing-and-bytecode.md) — lazy parsing, Ignition bytecode, the register machine
- [JIT Compilation](engine-internals/jit-compilation.md) — the tiers: Ignition → Sparkplug → Maglev → TurboFan
- [Hidden Classes & Inline Caches](engine-internals/hidden-classes-and-inline-caches.md) — object shapes, monomorphism, ICs
- [Deoptimization](engine-internals/deoptimization.md) — bailouts, deopt loops, staying optimizable

## Language Semantics
- [Type System & Coercion](language-semantics/type-system-and-coercion.md)
- [Prototypes & Inheritance](language-semantics/prototypes-and-inheritance.md)
- [`this` & Binding](language-semantics/this-and-binding.md)
- [Scopes & Closures](language-semantics/scopes-and-closures.md)
- [Property Descriptors & Proxies](language-semantics/property-descriptors-and-proxies.md)
- [Iterators & Generators](language-semantics/iterators-and-generators.md)
- [Symbols](language-semantics/symbols.md)

## Modules
- [Module Systems](modules/module-systems.md) — globals → CJS → ESM, and why
- [ESM Internals](modules/esm-internals.md) — the parse/link/evaluate lifecycle, live bindings
- [Module Resolution](modules/module-resolution.md)

## Asynchrony
- [Promises Internals](asynchrony/promises-internals.md) — states, the microtask queue, thenable assimilation
- [Async / Await](asynchrony/async-await.md) — desugaring to a generator + driver state machine
- [Generators as Coroutines](asynchrony/generators-as-coroutines.md)

## Concurrency
- [Web Workers](concurrency/web-workers.md) — true parallelism, message passing, the actor model
- [Shared Memory & Atomics](concurrency/shared-memory-and-atomics.md) — `SharedArrayBuffer`, the JS memory model

## Self-Test
- [Review Questions](review-questions.md)
