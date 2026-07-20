# Bytecode and Class Files

A `.class` file is the JVM's portable executable format: a rigidly specified binary that packages one type's constant data, structure, and verified bytecode. It solves the "write once, run anywhere" problem by targeting an *abstract* stack machine rather than any real CPU, so the same bytes execute on any conformant JVM after a per-platform loader bridges them to hardware.

## Anatomy of a Class File

The format is a single, order-sensitive byte stream with no padding. Every file opens with the magic number `0xCAFEBABE`, then `minor_version`/`major_version`. The major version encodes the compiling JDK (Java N ≈ `44 + N`, so Java 8 is 52, Java 21 is 65, Java 25 is 69); a JVM refuses class files newer than itself. A `minor_version` of `0xFFFF` marks a **preview-feature** class, runnable only with `--enable-preview` on the exact matching release.

```text
magic  minor  major  cp_count  constant_pool[]
access_flags  this_class  super_class
interfaces[]  fields[]  methods[]  attributes[]
```

- **Constant pool** — the file's symbol table, 1-indexed (index 0 is reserved; `long`/`double` entries perversely consume *two* slots). Tagged entries hold UTF-8 strings, numeric literals, and *symbolic references*: `CONSTANT_Class`, `CONSTANT_Methodref`, `CONSTANT_NameAndType`. Bytecode never names things directly — it points into the pool, which is what makes references resolvable and relocatable.
- **access_flags** — a bitmask (`ACC_PUBLIC` `0x0001`, `ACC_FINAL` `0x0010`, `ACC_INTERFACE` `0x0200`, `ACC_ABSTRACT` `0x0400`, `ACC_SYNTHETIC`…) describing the type's modifiers.
- **fields / methods** — each has its own flags, name/descriptor pool indices, and attributes.
- **attributes** — the extensible tail. The critical one is a method's `Code` attribute, carrying `max_stack`, `max_locals`, the raw instruction array, an exception table, and nested attributes (`LineNumberTable`, `LocalVariableTable`, and the verification-critical `StackMapTable`).

## The Stack-Based Instruction Set

The JVM is a **stack machine**: instructions consume and produce values on a per-frame *operand stack* rather than addressing named registers. `iload_1` pushes local slot 1; `iadd` pops two ints and pushes their sum; `invokevirtual #7` pops the receiver and arguments, dispatches through pool entry 7, and pushes the result. Opcodes are one byte (hence ~200 max), type-prefixed (`i`/`l`/`f`/`d`/`a` for int/long/float/double/reference). Five invoke forms carry the method model: `invokevirtual` (virtual dispatch), `invokestatic`, `invokespecial` (constructors, `super`, private), `invokeinterface`, and `invokedynamic` — the user-definable call whose target is computed once by a bootstrap method, underpinning lambdas, records' `equals`/`hashCode`, and string concatenation.

## How `javac` Lowers Source

`javac` does little optimization — it *desugars*. Generics erase to casts plus `checkcast` and synthetic **bridge methods**; enhanced-`for` becomes an iterator loop; lambdas compile to an `invokedynamic` bound to `LambdaMetafactory`; string `+` becomes `invokedynamic` via `StringConcatFactory` (Java 9+); `switch` on strings expands into `hashCode` plus `equals`; try-with-resources unfolds into `try/finally` with suppressed-exception handling. The heavy optimization is deliberately left to the runtime, where profile data exists.

## Verification

Before any bytecode runs, the **verifier** proves it is type-safe and structurally sound: the operand stack never under/overflows, types match each instruction, control flow stays in bounds, and `final`/access rules hold. Modern class files ship `StackMapTable` frames so verification is a linear type-*check* rather than an expensive fixpoint inference (the old inferencing verifier survives only for pre-Java-6 files). This front-loaded proof is what lets HotSpot omit runtime type checks the JIT would otherwise need — safety bought once, at load time.

## What Portable Bytecode Buys — and the V8 Contrast

Bytecode is an intermediate representation, not source and not native code: compact, quick to load, and *pre-verified*, while still abstract enough to run anywhere and to be JIT-compiled with runtime profile data an AOT compiler lacks. V8 also compiles JS to bytecode, but its **Ignition** interpreter is *register-based* — an accumulator plus a register file — chosen to shrink bytecode size and interpreter dispatch cost. The JVM's stack model trades slightly denser code and simpler verification for more `load`/`store` traffic that the JIT later flattens into registers anyway. See [[01-programming-foundations/languages/javascript/engine-internals/parsing-and-bytecode|JS Parsing & Bytecode]].

## Senior Pitfalls

- Reading `.class` bytes as "what runs" — the JIT reshapes them beyond recognition; bytecode is the *input* to optimization, not the executed form.
- Assuming `long`/`double` occupy one constant-pool slot or one local slot; both take two, a classic off-by-one in bytecode tooling.
- Treating a `major_version` mismatch (`UnsupportedClassVersionError`) as a code bug rather than a toolchain-vs-runtime version skew.

## See Also

- [[jvm-architecture]]
- [[execution-engine-and-jit]]
- [[class-loading]]
- [[escape-analysis-and-optimizations]]
- [[01-programming-foundations/languages/javascript/engine-internals/parsing-and-bytecode|JS Parsing & Bytecode]]
