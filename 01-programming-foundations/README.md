# 01 · Programming Foundations

The conceptual bedrock: the paradigms that shape how we structure programs, and the internals of the language runtime used throughout this vault (JavaScript).

[← Back to vault index](../README.md)

## Programming Paradigms

### Core paradigms
- [Imperative Programming](paradigms/imperative-programming.md)
- [Procedural Programming](paradigms/procedural-programming.md)
- [Object-Oriented Programming](paradigms/object-oriented-programming.md)
- [Functional Programming](paradigms/functional-programming.md)
- [Declarative Programming](paradigms/declarative-programming.md)
- [Logic Programming](paradigms/logic-programming.md)
- [Reactive Programming](paradigms/reactive-programming.md)
- [Event-Driven Programming](paradigms/event-driven-programming.md)
- [Dataflow Programming](paradigms/dataflow-programming.md)
- [Aspect-Oriented Programming](paradigms/aspect-oriented-programming.md)

### Concepts & techniques
- [Pure Functions](paradigms/pure-functions.md)
- [Immutability](paradigms/immutability.md)
- [Higher-Order Functions](paradigms/higher-order-functions.md)
- [Closures](paradigms/closures.md)
- [Monads and Functors](paradigms/monads-and-functors.md)
- [Recursion and Tail Calls](paradigms/recursion-and-tail-calls.md)
- [Pattern Matching](paradigms/pattern-matching.md)
- [Type Systems](paradigms/type-systems.md)
- [Generics and Polymorphism](paradigms/generics-and-polymorphism.md)
- [Inheritance vs Composition](paradigms/inheritance-vs-composition.md)

## Languages

### JavaScript

A deep track on the language and its runtime — **full index: [JavaScript](languages/javascript/README.md).**

Execution model (event loop, GC) · Engine internals (V8 pipeline, JIT tiers, hidden classes, deopt) · Language semantics (coercion, prototypes, `this`, closures, proxies, generators, symbols) · Modules (ESM internals & resolution) · Asynchrony (promises internals, async/await desugaring) · Concurrency (web workers, shared memory & atomics).

### Java

A deep track on the language + JVM — **full index: [Java](languages/java/README.md).**

Type system & semantics (generics/erasure, variance, records, sealed types, pattern matching, lambdas) · The JVM (bytecode, class loading, C1/C2 JIT, escape analysis, safepoints) · Java Memory Model (happens-before, `volatile`, safe publication) · Garbage collection (G1, ZGC/Shenandoah, tuning) · Concurrency (executors, fork/join, locks & atomics, **virtual threads**, structured concurrency) · Modules & runtime (JPMS, reflection & method handles, VarHandle, jlink) · I/O (java.io, NIO, the Stream API) · Philosophy (incl. a [Java vs JavaScript](languages/java/philosophy/java-vs-javascript.md) contrast). Scoped to the language/runtime core; frameworks (Spring etc.) are a follow-up.
