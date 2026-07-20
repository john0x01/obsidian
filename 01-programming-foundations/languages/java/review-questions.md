# Java Review Questions

Senior-level self-test for the Java track. Questions only — answer from your own mental model, then check against the linked notes. Grouped by domain. Many are framed to surface the *why* and the JavaScript contrast.

## Type System & Language Semantics

1. Java generics are erased — what exactly is erased, what survives at runtime, and why can't you write `new T[]` or `instanceof List<String>`? What are bridge methods?
2. Explain covariance, contravariance, and invariance in Java. When do you reach for `? extends T` vs `? super T` (PECS)?
3. What problem do records solve, and what do they generate? How do sealed types enable exhaustive pattern matching without a `default`?
4. Walk the `equals`/`hashCode` contract. What breaks if you override one but not the other, or use a mutable field in `hashCode` as a `HashMap` key?
5. Checked vs unchecked exceptions: what is the compiler's rule, what is the long-running critique, and what's modern practice (esp. with lambdas/streams)?
6. How do lambdas compile? Why `invokedynamic` + `LambdaMetafactory` rather than anonymous inner classes, and when is a lambda instance cached vs created per-capture?
7. What does a lambda capture, and how does "effectively final" differ from a JS closure capturing a live binding?
8. Why is `Optional` a return type, not a field type? What are the anti-patterns (`Optional.get`, `Optional` fields, `isPresent`+`get`)?
9. Autoboxing: what is the integer cache, and why can `==` on two `Integer` values holding 1000 be `false`?

## The JVM

1. Trace a method from source to native: `javac` → class file → verification → interpreter → C1 → C2. What triggers each tier?
2. What is in a class file (constant pool, attributes, `StackMapTable`)? Why is the JVM a stack machine, and how does that differ from V8's register-based Ignition?
3. What is on-stack replacement (OSR), and why is it needed for long-running loops?
4. What is a deoptimization / uncommon trap? Give patterns that cause one.
5. Explain escape analysis → scalar replacement and lock elision. Why is "objects are always heap-allocated" not quite true, and why do naive microbenchmarks mislead here?
6. Describe class loading: the loader delegation model, when a class is initialized vs loaded, and how class identity depends on `(name, classloader)`.
7. What is a safepoint? Explain time-to-safepoint, why a counted loop can inflate it, and how safepoint bias makes some profilers lie.
8. What does GraalVM change (JIT via JVMCI, and native-image AOT with its closed-world assumption)?

## Java Memory Model

1. State the happens-before relationship. What orderings do program order, monitor lock/unlock, and `volatile` write/read establish?
2. What exactly does `volatile` guarantee — and what does it *not* (hint: it's not a mutex)?
3. What is a data race vs a race condition? What is the defined behavior of a data race in Java (contrast with C++ UB and with the JS single-threaded model)?
4. Explain the `final`-field freeze: why can a correctly constructed immutable object be safely published without synchronization, and what breaks if `this` escapes the constructor?
5. Enumerate the safe-publication idioms (static initializer, `volatile`, `final`, concurrent collection, locking).

## Garbage Collection

1. Compare G1, ZGC, Shenandoah, Parallel, Serial, and Epsilon — what is each optimized for, and how do you choose?
2. How does G1's region model + remembered sets + SATB marking achieve a pause-time goal? What are humongous objects and why are they a hazard?
3. How do ZGC and Shenandoah compact *concurrently*? What role do load/load-reference barriers and colored pointers play, and what is the throughput cost?
4. What still happens at a safepoint even in a "pauseless" collector?
5. How do you diagnose high allocation rate and GC pressure? What flags and logs do you start with?
6. Contrast Java's GC control surface with V8's opaque Orinoco — what can you tune in Java that you can't in Node?

## Concurrency

1. How does a `ThreadPoolExecutor` decide to use the core pool, the queue, or spawn up to max? Why is an unbounded `LinkedBlockingQueue` a silent trap?
2. How does the fork/join pool's work-stealing work (LIFO own / FIFO steal)? Why can parallel streams starve on the common pool, and when do they hurt more than help?
3. Compare `synchronized`, `ReentrantLock`, `ReadWriteLock`, and `StampedLock`. What is AQS and how does it back most of them?
4. Explain CAS and the ABA problem. When do you use `LongAdder` over `AtomicLong`?
5. How does `ConcurrentHashMap` allow lock-free reads and per-bin writes? What is a weakly-consistent iterator, and why does the map forbid `null`?
6. What is a virtual thread, mechanically (continuations, carrier threads, mount/unmount)? Why is pooling them an anti-pattern, and what causes pinning?
7. Why do virtual threads make "async" function coloring unnecessary? Contrast the thread-per-request model with the Node event loop under a slow synchronous call.
8. What does structured concurrency give you over `ExecutorService` + `Future`? What are scoped values and why do they replace `ThreadLocal` for virtual threads?
9. Map `CompletableFuture` combinators to JS Promise methods. Where does CF run a non-`Async` stage, and how is that different from JS's always-async `.then`?

## Modules & Runtime

1. What does JPMS enforce that the classpath cannot? What is strong encapsulation, and how do `requires`/`exports`/`opens` differ?
2. What are the three structural costs of reflection, and how do method handles + `invokedynamic` avoid them?
3. What are VarHandle access modes (plain/opaque/acquire-release/volatile), and how do they map onto the memory model? Why is VarHandle the sanctioned replacement for `Unsafe`?
4. What does jlink produce, and how do AppCDS/CDS and jpackage attack JVM startup and footprint?

## I/O

1. Byte streams vs character streams vs the functional Stream API — what are the three different "streams", and how do you keep them straight?
2. What does the decorator pattern buy `java.io`, and why is `BufferedReader` around a `FileReader` idiomatic?
3. What does NIO add over `java.io` (buffers, channels, selectors)? How does a selector-based reactor map onto epoll/kqueue, and how does that compare to libuv?
4. Why are Stream pipelines lazy, and what does that enable (short-circuiting, fusion)? When does `.parallel()` actually pay off?
5. How do virtual threads change the "blocking I/O doesn't scale" calculus that motivated NIO and reactive stacks?

## Design & Philosophy

1. Why does Java prize backward compatibility so fiercely, and what does that cost the language? Contrast with the JS/transpiler churn model.
2. Where does Java's "explicit and verbose" instinct help large teams, and where does it hurt? Pick three places modern Java (records, `var`, pattern matching) softened it.
3. Why did the JVM outgrow Java-the-language as the real moat? What made it a good polyglot target?
