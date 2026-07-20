# Class Loading

Class loading is how the JVM turns a `.class` file's bytes into a usable runtime type, *on demand*. It solves a subtle problem: a program is a graph of types whose dependencies aren't known until execution reaches them, and which may come from disk, the network, or be generated at runtime. The JVM defers, verifies, and isolates this work through a well-defined lifecycle and a delegating hierarchy of loaders.

## The Lifecycle

```text
Loading → Linking(Verify → Prepare → Resolve) → Initialization
```

- **Loading** — a class loader finds the binary, parses it into an internal representation, and creates the `Class` object in metaspace.
- **Linking**
  - *Verify* — bytecode verification proves the code is type-safe and structurally sound (operand-stack consistency via `StackMapTable` frames) before it ever runs. This is the JVM's safety backbone; it lets HotSpot skip many runtime checks.
  - *Prepare* — allocates static fields and sets them to **default values** (`0`, `null`, `false`) — *not* their declared initializers.
  - *Resolve* — replaces symbolic references in the constant pool with direct references. Can be lazy.
- **Initialization** — runs `<clinit>`, the synthesized static initializer that executes static-field initializers and `static {}` blocks, *in source order*. This is the moment user code first runs.

## Lazy Initialization Triggers

A class is initialized only on first *active use*: instantiation (`new`), invoking/accessing a static method or non-constant static field, reflection, or initializing a subclass. Crucially, reading a `static final` **compile-time constant** does *not* trigger initialization — the value was inlined into callers at compile time. The JVM guarantees `<clinit>` runs exactly once and is thread-safe; the classloader holds an initialization lock, which is the mechanism behind the lazy-holder (initialization-on-demand holder) idiom for lazy singletons.

## The Loader Hierarchy and Parent Delegation

Three built-in loaders form a parent chain:

- **Bootstrap** — written in native code; loads core `java.*` classes. Its parent is conceptually `null`.
- **Platform** — loads JDK modules beyond the core (e.g., certain `java.*`/`jdk.*` services).
- **Application (system)** — loads your classpath/module-path code.

Under **parent delegation**, a loader asks its *parent* first and only loads the class itself if every ancestor fails. This guarantees that core types load from the trusted bootstrap loader — application code can't shadow `java.lang.String` with a malicious copy — and avoids duplicate core classes.

## Class Identity = (Binary Name, Defining Loader)

Two `Class` objects with the same name loaded by *different* loaders are **distinct types**. Casting between them throws `ClassCastException`; a field of one can't hold an instance of the other. This is the foundation of **isolation**: app servers and plugin systems give each module its own loader so libraries with the same class name don't collide. Custom loaders (extend `ClassLoader`, override `findClass`) enable hot-reload (load a new version under a fresh loader), sandboxing, and generated/encrypted bytecode via `defineClass`.

## CNFE vs. NoClassDefFoundError

A constant source of confusion:

- `ClassNotFoundException` — a *checked* exception thrown when explicit loading (`Class.forName`, `loader.loadClass`) can't find the class. The class was never successfully present.
- `NoClassDefFoundError` — an *Error* thrown when a class that linked *successfully at compile time* is absent at runtime, **or** when a class's `<clinit>` previously threw (leaving it in a failed state — subsequent uses get `NoClassDefFoundError`, masking the original `ExceptionInInitializerError`). The classic gotcha: blaming a missing jar when the real cause is an exception in static initialization.

## Unloading

A class can be unloaded only when its *defining loader* becomes unreachable and GC-collectible — classes and their loader live and die together. A leaked loader (e.g., a thread-local or static reference held by a parent loader pinning child-loaded classes) causes the textbook `OutOfMemoryError: Metaspace` in redeploy-heavy servers.

## Contrast with JavaScript

JS modules (ESM) resolve and load *before* execution: the loader fetches the dependency graph, instantiates bindings, then evaluates — eager and link-time. The JVM is the opposite: lazy, per-type, triggered by first use, with verification and an isolation model (loader identity) that ESM has no equivalent for. Both share the idea of a module graph; only the JVM makes the *loader* part of a type's identity.

## See Also

- [[jvm-architecture]]
- [[bytecode-and-class-files]]
- [[jpms-module-system]]
- [[reflection-and-method-handles]]
- [[01-programming-foundations/languages/javascript/modules/module-systems|JS Module Systems]]
