# Generics and Type Erasure

Java generics give compile-time type safety for parameterized types (`List<String>`), but they are implemented by **type erasure**: the type parameters exist only at compile time and are *erased* before bytecode is produced. The runtime sees `List`, not `List<String>`. Understanding erasure explains almost every "weird" generics rule you'll ever hit.

## Why Erasure? Migration Compatibility

Generics arrived in Java 5, a decade into the language. Billions of lines already used raw `List`, `Map`, etc. The design constraint was **binary backward compatibility**: pre-generics code and generic code had to interoperate, and old `.class` files had to keep working unchanged. Erasure achieves this — `List<String>` and the old raw `List` are the *same type* at runtime, so a generic `List<String>` can be passed to legacy code expecting `List`, and vice versa. The price: **zero runtime type information** about type arguments. C# took the opposite road with **reified** generics (the CLR knows `List<int>` at runtime), but C# broke compatibility to do it — a luxury Java didn't have.

## What Erasure Actually Does

At compile time, the compiler:

1. Replaces each type parameter with its **leftmost bound** (`T` → `Object`; `<T extends Number>` → `Number`).
2. Inserts **synthetic casts** at call sites where the erased type is used.
3. Generates **bridge methods** where needed (below).

```java
// You write:
class Box<T> { T get() { return value; } T value; }
String s = box.get();
// Erased to:
class Box { Object get() { return value; } Object value; }
String s = (String) box.get();   // compiler-inserted cast
```

So generics are, in a real sense, **compiler-enforced casting discipline** — the runtime is exactly as it was pre-generics.

## The Reification Gap

Because no runtime type-argument info exists, several operations are simply **impossible**:

- **`new T[]`** — illegal. The array would need a runtime class to enforce `ArrayStoreException`, but `T` is erased. You're forced into `(T[]) new Object[n]` with an unchecked warning.
- **`new T()`** — illegal; no way to call the right constructor.
- **`obj instanceof List<String>`** — illegal. At runtime there's only `List`; you can only test `instanceof List<?>`.
- **No overloading on erased signatures**: `void f(List<String>)` and `void f(List<Integer>)` collide — both erase to `f(List)`. Compile error: "same erasure."
- **No `T.class`**, no generic exception classes, no `static` fields of type `T`.

This is the **reification gap**: the type checker pretends types are real, but the runtime forgot them.

## Bridge Methods

Erasure breaks polymorphism for generic supertypes and covariant returns, so the compiler patches it with **synthetic bridge methods**. Consider:

```java
class StringBox implements Comparable<StringBox> {
    public int compareTo(StringBox o) { ... }
}
```

`Comparable<StringBox>` erases to `Comparable` with `compareTo(Object)`. But `StringBox` only declares `compareTo(StringBox)`. To honor the interface's erased contract, the compiler generates a bridge:

```java
public int compareTo(Object o) {        // synthetic bridge
    return compareTo((StringBox) o);    // delegates, casts
}
```

Bridge methods are why a generic override "just works" through an erased interface reference, and why reflection sometimes shows two `compareTo` methods (one flagged `isBridge()`).

## Heap Pollution, Unchecked Warnings, `@SafeVarargs`

Because casts are unchecked, you can smuggle a wrong type into a generic container — **heap pollution**: the static type claims `List<String>` but a non-`String` actually sits inside. The mismatch surfaces later as a `ClassCastException` from a compiler-inserted cast, far from the real cause. The compiler flags risky spots with **unchecked warnings** — never blanket-suppress them; suppress narrowly and prove safety.

Generic varargs are a notorious source: `T...` compiles to `T[]`, which (per the reification gap) can't be created safely, so the compiler warns at every call. If your method genuinely only *reads* the varargs array and never lets it escape, annotate it `@SafeVarargs` to vouch for it and silence callers.

## Senior Takeaways

- Generics are a **compile-time fiction** layered over an untyped runtime — design APIs knowing the runtime can't help you.
- When you need runtime type info, pass a **`Class<T>` token** explicitly (the "type token" pattern, as `Class.cast` / `EnumSet.noneOf` do).
- Don't fight erasure expecting C#-style reification; embrace bounds, wildcards, and tokens instead.

## See Also

- [[variance-and-wildcards]]
- [[type-system-and-conversions]]
- [[reflection-and-method-handles]]
- [[01-programming-foundations/paradigms/generics-and-polymorphism|Generics & Polymorphism]]
- [[01-programming-foundations/paradigms/type-systems|Type Systems]]
