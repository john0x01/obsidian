# Reflection And Method Handles

Reflection lets a running program inspect and invoke types it did not know at compile time; method handles are the faster, JIT-friendly successor that the platform itself now runs on. Together they are how frameworks — dependency injection, serializers, ORMs, mocking libraries — bend a statically-typed language into something dynamically extensible, and understanding their cost model is what separates a framework author from a framework user.

## The Reflection API

`java.lang.reflect` exposes the runtime metamodel: a `Class<?>` yields `Method`, `Field`, `Constructor`, and annotation objects, each queryable and invocable. The pivotal capability is `AccessibleObject.setAccessible(true)`, which suppresses Java's access checks so a framework can read a `private` field or call a `private` method.

Under JPMS this is no longer a free pass. `setAccessible` succeeds only if the target package is *open* — via an `opens` directive in `module-info.java`, an `--add-opens` flag, or an automatic/unnamed module. Against a strongly-encapsulated module it throws `InaccessibleObjectException`. This is exactly the `exports`-vs-`opens` split from [[jpms-module-system]]: exporting grants normal access; only opening grants *deep reflective* access. It is why Jackson and Hibernate ask you to `opens` your model packages.

## The Cost Of Reflection

Reflective invocation is slow for structural reasons, not incidental ones:

- **Access and security checks** on each call (mitigated by caching an `AccessibleObject` you have already made accessible).
- **Argument boxing**: `Method.invoke(Object, Object...)` forces primitives into wrapper objects packed in a varargs array, with matching unboxing on return.
- **Opacity to the JIT**: a reflective target is generally not a compile-time constant, so it cannot be inlined; the optimizer sees an opaque call.

Historically the JDK also *inflated* hot reflective calls — after roughly fifteen invocations it spun a bespoke bytecode accessor class to replace the slow native path (the old `sun.reflect` machinery). JDK 18 reimplemented core reflection on top of method handles — `Method`, `Field`, and `Constructor` now delegate to a `MethodHandle` — retiring the bytecode-spinning inflation entirely and letting reflection ride the same optimizations as everything else.

## Method Handles

A `MethodHandle` (`java.lang.invoke`, JSR-292, Java 7) is a *typed, directly-executable* reference to a method, field accessor, or constructor. You obtain one from a `MethodHandles.Lookup` — a capability object capturing the *lookup class's* access rights at the moment it is created — via `findVirtual`, `findStatic`, `findGetter`, and so on. Access is checked **once**, at lookup time, not per call; `privateLookupIn` lets a cooperating module hand out private access.

The performance win is that a `MethodHandle` stored in a `static final` field is treated by the JIT as a constant, so calls through it inline as if hand-written. `invokeExact` demands the call-site signature match the handle's `MethodType` exactly (no boxing, no widening); `invoke` permits `asType` adaptation. Handles also compose — `filterArguments`, `insertArguments`, `guardWithTest` — building small dynamic call graphs without generating classes.

```
reflection:   Method.invoke → checks + box args → opaque call
methodhandle: constant MH    → checked once      → inlined by JIT
```

## invokedynamic

`invokedynamic` (Java 7) is the bytecode instruction that defers *linking* a call site to runtime. On first execution the JVM calls a **bootstrap method**, which returns a `CallSite` wrapping a `MethodHandle`; that target is then linked and every later execution jumps straight to it. It made the JVM a first-class host for dynamic languages, but two mainstream Java features now depend on it:

- **Lambdas.** `javac` does not emit an anonymous class per lambda. It emits an `invokedynamic` whose bootstrap is `LambdaMetafactory.metafactory`, which at runtime spins the functional-interface implementation (a hidden class). This keeps class count and jar size down and lets the strategy evolve without recompiling.
- **String concatenation** (JEP 280, Java 9). The `+` operator on strings compiles to an `invokedynamic` bound to `StringConcatFactory`, not a hard-coded `StringBuilder` chain — so the runtime chooses the concat strategy and can presize the result.

Records reuse the same mechanism: their `equals`/`hashCode`/`toString` are generated through `java.lang.runtime.ObjectMethods` via `invokedynamic` (see [[records-and-sealed-types]]). The through-line is philosophical: push work from *compile time* to *link time*, keep bytecode small and stable, and let the JIT specialize constant method handles at peak speed — the same "trust the runtime" bet behind [[07-performance-engineering/jit-vs-aot|JIT vs AOT]].

## See Also
- [[jpms-module-system]]
- [[class-loading]]
- [[varhandle-and-unsafe]]
- [[records-and-sealed-types]]
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]
