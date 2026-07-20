# Java vs JavaScript

For an engineer fluent in JavaScript, the fastest route to Java mastery is to map each mental model onto its Java counterpart and — more importantly — to notice where the models *diverge*. The names rhyme for historical marketing reasons; the languages are built on opposite instincts. JavaScript optimizes for flexibility and iteration speed; Java optimizes for large-team correctness and long-term evolution.

## Type System

JavaScript is **dynamically, weakly typed**: types live on values, coercion is pervasive, and structure is duck-typed. Java is **statically, strongly, nominally typed**: types live on variables/expressions, are checked at compile time, and identity is by *declared name* (implementing an interface is explicit, not structural). Java generics are **erased** (compile-time only, like TypeScript's) — but unlike TS, Java's checks are enforced by `javac` and the verifier, and there is no gradual `any` escape hatch. The trade is classic: JS lets you move fast and discover shape at runtime; Java makes the compiler a design tool and refactoring engine.

See [[type-system-and-conversions]] and [[01-programming-foundations/languages/javascript/language-semantics/type-system-and-coercion|JS Type System & Coercion]].

## Execution & Runtime

Both compile to bytecode and JIT to native — V8 (Ignition → Sparkplug → Maglev → TurboFan) and HotSpot (interpreter → C1 → C2). Two structural differences dominate:

- **Warmup vs startup.** The JVM is a long-lived server runtime: it interprets, profiles, then tiers up to C2 over seconds — great for throughput, historically bad for CLI/serverless cold starts (mitigated by AppCDS, GraalVM native-image AOT). V8 is tuned for fast startup and short-lived page loads.
- **Speculation.** Both speculate on types and deoptimize on violation, but V8 leans on hidden classes/inline caches because *any* object shape can change; the JVM's nominal types make monomorphic dispatch the common case.

## Concurrency — the biggest divergence

JavaScript is **single-threaded with an event loop**: no shared-memory data races by construction, concurrency via callbacks/promises/`async`, parallelism only via Workers that communicate by copying messages. Java is **genuinely multi-threaded with shared memory**: real OS threads, a formal **Java Memory Model** with `happens-before`, `volatile`, and `synchronized`, and now **virtual threads** (Project Loom) that make cheap blocking I/O concurrency the default. The mental shift is large:

- In JS you never worry about two threads touching one object; in Java that's the central hazard (see [[java-memory-model]]).
- JS "async colors" every function (`async`/`await` propagates); Java virtual threads let ordinary *blocking* code scale to millions of tasks — **no function coloring** (see [[virtual-threads]]).
- A slow synchronous computation stalls the *entire* JS event loop; in Java it occupies one thread while others proceed.

Contrast [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime & Event Loop]].

## Memory & GC

Both are tracing, generational, and non-deterministic — you don't `free`. The difference is *control and scale*: the JVM exposes a menu of collectors (G1, ZGC, Shenandoah, Parallel) and extensive tuning for multi-gigabyte heaps and latency SLAs; V8's GC (Orinoco) is largely opaque and tuned for browser/Node workloads. Leak shapes rhyme (lingering references, caches, listeners); Java adds `WeakReference`/`ThreadLocal` pitfalls, JS adds detached-DOM leaks.

## Objects, Functions, Modules

- **Objects:** Java is class-based with nominal inheritance + interfaces; JavaScript is prototype-based with delegation. "Composition over inheritance" is advice in both, but Java gives you sealed types and records for closed, data-centric modeling.
- **Functions:** first-class and closure-native in JS since day one; Java retrofitted them as **lambdas** compiled via `invokedynamic`/`LambdaMetafactory` (not anonymous classes), capturing *effectively-final* values *by value* — versus JS closures capturing live bindings (see [[lambdas-and-functional-interfaces]]).
- **Modules:** JS `import`/ESM resolves (often dynamically) at load; Java has two layers — the classpath and the stronger, compile-time **JPMS** module system with real encapsulation (see [[jpms-module-system]]).

## Error Handling & Philosophy

Java has **checked exceptions** — the compiler forces you to declare or handle them, a feature unique among mainstream languages and endlessly debated (see [[exceptions-and-error-handling]]). JS throws anything, anywhere, unchecked. This encapsulates the worldviews: Java pushes failure modes into the type system and prizes **backward compatibility** (decade-old bytecode still runs); JavaScript prizes **flexibility** and ships breaking idioms via transpilers. Neither is "better" — they're tuned for different failure costs. Knowing *which instinct* a language follows is what lets you write idiomatic code in it rather than JavaScript-in-Java.

## See Also

- [[design-philosophy]]
- [[java-memory-model]]
- [[virtual-threads]]
- [[lambdas-and-functional-interfaces]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime & Event Loop]]
- [[01-programming-foundations/languages/javascript/language-semantics/type-system-and-coercion|JS Type System & Coercion]]
