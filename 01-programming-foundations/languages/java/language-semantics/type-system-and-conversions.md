# Type System and Conversions

Java has a **static, nominal, manifestly-typed** type system: every expression has a type known at compile time, types relate by *name* (not structure), and types are checked before the program runs. This is the polar opposite of JavaScript's dynamic, structural-ish, coercion-happy model, and it's the source of most of Java's safety guarantees — and a few of its sharpest pitfalls.

## Two Worlds: Primitives and References

Java's type universe splits into two disjoint regions:

- **8 primitives**: `boolean`, `byte`, `short`, `char`, `int`, `long`, `float`, `double`. These are *values*, not objects. They live directly in a local variable slot or inline in an object's field layout — no header, no identity, no `null`.
- **Reference types**: classes, interfaces, arrays, enums. A variable holds a *reference* (effectively a pointer) to a heap object, or `null`.

Primitives sit **outside the `Object` hierarchy** deliberately, for performance. An `int` is 4 bytes with zero indirection; boxing it as an object would add a header (~12–16 bytes) plus a pointer dereference per access. The early designers traded uniformity for raw arithmetic speed — a defensible call in 1995, and the gap Project Valhalla now aims to close.

Everything else is **unified under `java.lang.Object`**: every reference type ultimately extends `Object`, so any object can go where an `Object` is expected. Arrays are objects too. Primitives are the *only* exception to "everything is an Object," which is why generics can't hold them directly.

## Wrappers, Autoboxing, and Its Hazards

Each primitive has a wrapper class (`Integer`, `Long`, `Boolean`, `Character`, …). **Autoboxing** silently converts `int → Integer`; **unboxing** converts back. The compiler inserts `Integer.valueOf(x)` and `intValue()` calls for you. Convenient — and a trap.

```java
Integer a = 127, b = 127;
a == b;            // true  — same cached object
Integer c = 128, d = 128;
c == d;            // false — distinct objects
```

The **`Integer` cache** (`−128..127` by default) means `valueOf` returns interned instances in that range, so `==` *accidentally* works for small values and breaks above 127. `==` on references is **identity** (same object), never value. Always use `.equals` for wrapper equality.

Two more hazards:

- **NPE on unboxing `null`**: `Integer x = map.get("missing"); int y = x;` throws `NullPointerException` — the unbox call `x.intValue()` dereferences `null`. This is invisible in source.
- **Mixed comparisons**: `Integer == int` unboxes the wrapper, so `==` becomes a value compare there — inconsistent with `Integer == Integer`.

Prefer primitives in hot paths; reach for wrappers only when you need nullability or generics.

## Conversions and Numeric Promotion

- **Widening** (`int → long → float → double`) is implicit and *lossless in range* (though `long → float`/`double` can lose precision).
- **Narrowing** (`long → int`, `double → int`) requires an explicit cast and silently truncates/overflows.
- **Binary numeric promotion**: in arithmetic, operands narrower than `int` are promoted to `int` first. So `byte + byte` yields an `int`; `byte b = b1 + b2;` won't compile without a cast. If either operand is `long`/`float`/`double`, both promote to the widest.

This is why `1 / 2 == 0` (integer division) but `1.0 / 2 == 0.5`.

## `var`: Inference, Not Dynamism

`var` (Java 10+) is **compile-time local type inference**, not dynamic typing. `var list = new ArrayList<String>();` infers `ArrayList<String>` *at compile time* and locks it in — the variable's static type is fixed forever; you cannot reassign a `String` to it. It's pure syntactic relief for obvious right-hand sides. It works only for local variables with initializers, not fields, params, or return types — keeping the system manifestly typed at API boundaries.

## Future Direction: Valhalla

Project **Valhalla** (value/primitive classes, null-restricted types) aims to give the JVM "codes like a class, works like an int" — flattened, identity-free value objects that erase the primitive/reference divide. It is **not shipped**; treat it as a roadmap, not a tool.

## Senior Mental Model

Contrast JS: there, `"5" - 1 === 4` and `1 + "1" === "11"` — coercion is implicit and lossy by *default*. Java refuses almost all of this; `"a" + 1` concatenates only because `+` is overloaded for `String`, and there's no `-` on strings at all. Java pushes errors to compile time; JS defers them to runtime surprises. The cost is ceremony; the payoff is that an `int` is always an `int`.

## See Also

- [[generics-and-erasure]]
- [[nullability-and-optional]]
- [[equality-and-contracts]]
- [[java-vs-javascript]]
- [[01-programming-foundations/languages/javascript/language-semantics/type-system-and-coercion|JS Type System & Coercion]]
- [[01-programming-foundations/paradigms/type-systems|Type Systems]]
