# JVM Architecture

The Java Virtual Machine is an *abstract computing machine* — a specification of an instruction set, a memory layout, and an execution model — not a single program. "The JVM" you run is an *implementation* of that spec; the dominant one is Oracle/OpenJDK **HotSpot**. This split is the central idea: the JVM Specification defines behavior a program can rely on, while implementations are free to use interpreters, JIT compilers, garbage collectors, and memory tricks the spec never mentions.

## Spec vs. Implementation

The JVM is a **stack machine**: instructions operate on an operand stack rather than named registers. The spec defines runtime data areas, the `.class` file format, and bytecode semantics — but says almost nothing about *how* they're realized. HotSpot interprets bytecode, then JIT-compiles hot methods to native code; it has a generational heap and multiple GCs. None of that is mandated. A conformant JVM could be a pure interpreter on a microcontroller. This is why "the JVM is slow because it's interpreted" is a category error: the *spec* is interpreted in concept; the *implementation* compiles aggressively.

## Runtime Data Areas

The spec partitions memory into per-thread and shared regions:

```text
        SHARED (all threads)                 PER-THREAD
  ┌───────────────────────────┐      ┌──────────────────────┐
  │ Heap (objects, arrays)     │      │ PC register          │
  │ Metaspace (class metadata) │      │ JVM stack (frames)   │
  │ Runtime constant pool      │      │ Native method stack  │
  │ Code cache (JIT output)    │      └──────────────────────┘
  └───────────────────────────┘
```

- **Heap** — shared, GC-managed; every object and array lives here. The one region the GC governs.
- **Metaspace** — off-heap *native* memory holding class metadata (the runtime form of `.class` data: method bytecode, field layouts, the runtime constant pool). This replaced **PermGen**, removed in Java 8; metadata now grows in native memory rather than a fixed heap region.
- **Code cache** — native memory where the JIT stores compiled machine code; if it fills, compilation stalls and you fall back to the interpreter.
- **PC register** (per-thread) — holds the address of the current bytecode instruction; undefined while executing a native method.
- **JVM stack** (per-thread) — a stack of **frames**, one per active method call. Overflow throws `StackOverflowError`.
- **Native method stack** (per-thread) — for native (JNI/FFM) calls.

## The Frame

Each method invocation pushes a frame with three core parts:

- **Local variable array** — indexed slots holding arguments and locals. For instance methods, slot 0 is `this`. `long`/`double` occupy two slots.
- **Operand stack** — the working scratchpad. `iload_1` pushes a local; `iadd` pops two and pushes the sum. The stack's max depth is fixed at compile time and recorded in the method's `Code` attribute.
- **Reference to the runtime constant pool** — so symbolic references (method/field/class names) can be resolved.

The frame is the unit of method-local state; this is why local variables are inherently thread-safe and why deep recursion overflows the stack rather than the heap.

## Bytecode and "Write Once, Run Anywhere"

`javac` compiles source to **bytecode** — the JVM's portable instruction set — not native code. Bytecode targets the *abstract* machine, so a single `.class` runs on any conformant JVM regardless of CPU or OS. Portability lives in the *intermediate* representation plus the per-platform JVM that bridges abstract instructions to real hardware. The cost: a startup penalty (loading, verifying, interpreting, then compiling). The payoff: one artifact, many platforms — and the freedom for HotSpot to optimize at runtime using information unavailable to an ahead-of-time compiler.

This mirrors a JavaScript engine: V8 parses JS to its own bytecode, interprets via Ignition, then JIT-compiles hot code via TurboFan — the same interpret-then-compile shape, with the source language itself as the portable artifact rather than a pre-compiled class file.

## Senior Pitfalls

- Conflating the heap with "all JVM memory." Stacks, metaspace, code cache, and thread structures are *off-heap*; a container OOM-kill with a healthy heap usually means native memory you forgot to account for.
- Assuming PermGen exists. `OutOfMemoryError: Metaspace` is the modern symptom of a classloader leak.
- Believing bytecode equals slow. After warmup, HotSpot's JIT routinely matches or beats naïve native code.

## See Also

- [[bytecode-and-class-files]]
- [[execution-engine-and-jit]]
- [[heap-and-allocation]]
- [[class-loading]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
