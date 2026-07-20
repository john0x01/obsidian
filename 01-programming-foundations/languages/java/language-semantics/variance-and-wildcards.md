# Variance and Wildcards

Variance is the answer to one question: if `Cat` is a subtype of `Animal`, what is the relationship between `List<Cat>` and `List<Animal>`? Java's answer — **invariance by default, opt-in covariance/contravariance via wildcards** — is one of the trickiest corners of the language, and getting it right is what separates fluent generic code from a wall of casts.

## Generics Are Invariant; Arrays Are Covariant

`List<Cat>` is **not** a subtype of `List<Animal>`, even though `Cat extends Animal`. This is **invariance**, and it's a *feature*. If it were allowed:

```java
List<Cat> cats = new ArrayList<>();
List<Animal> animals = cats;   // ILLEGAL — and here's why
animals.add(new Dog());        // would corrupt cats with a Dog
Cat c = cats.get(0);           // ClassCastException at read
```

Invariance closes that hole at compile time.

Arrays, however, are **covariant** — a 1995 design decision predating generics, made so generic-ish algorithms (like sorting) could work before generics existed:

```java
Object[] arr = new Cat[1];   // legal — covariant
arr[0] = new Dog();          // compiles fine, throws at RUNTIME
```

That last line throws **`ArrayStoreException`**. The JVM stores the array's true element type in its header and checks every write. So array covariance is *unsound* and pays for it with a runtime check on every store — a cautionary tale that directly motivated making generics invariant.

```
            covariant (arrays)      invariant (generics)
write:   runtime ArrayStoreException   compile-time error
cost:    per-write runtime check       zero runtime cost
```

## Use-Site Variance: Wildcards and PECS

Strict invariance is too rigid for real APIs, so Java adds **use-site variance** through wildcards. You declare the variance *where you use* the generic type, not where it's defined.

- **`? extends T` — covariant, a *producer***. You can **read** `T` out (everything is at least a `T`), but you **cannot write** anything in (the compiler can't know the exact subtype). `List<? extends Number>` could be a `List<Integer>` or `List<Double>` — so `add` is forbidden (except `null`), but `get` yields a `Number`.
- **`? super T` — contravariant, a *consumer***. You can **write** `T` (or subtypes) in, but reads come back only as `Object` (you can't know how far up the hierarchy the real type is). `List<? super Integer>` accepts `add(anInteger)`.

The mnemonic: **PECS — Producer Extends, Consumer Super**.

```java
// Copies FROM a producer INTO a consumer:
static <T> void copy(List<? extends T> src, List<? super T> dst) {
    for (T t : src) dst.add(t);
}
```

`src` produces `T`s (extends), `dst` consumes them (super). This signature accepts `copy(List<Integer>, List<Number>)` and many more combinations — far more flexible than a plain `List<T>` on both sides.

- **Unbounded `?`**: `List<?>` means "list of *some* unknown type." You can read elements as `Object` and call type-agnostic methods (`size`, `clear`), but cannot `add` anything except `null`. Use it when the type argument is irrelevant.

## Wildcard Capture

A wildcard is an *anonymous* type, but the compiler can **capture** it into a fresh synthetic type variable when it can prove consistency. This enables the "capture helper" idiom:

```java
static void reverse(List<?> list) { rev(list); }
private static <T> void rev(List<T> list) {  // captures ? as T
    // now T is a real, nameable type within this method
}
```

The public method takes a `List<?>`; the private generic helper *captures* that unknown into `T`, letting you write `set`/`get` against a consistent type. Capture errors ("`capture of ?`") almost always mean you tried to relate two independent wildcards that the compiler can't prove are the same type.

## Why Use-Site, Not Declaration-Site?

Languages like Scala/Kotlin/C# offer **declaration-site** variance (`out T`, `in T` / `? extends` baked into the type). Java chose **use-site** wildcards because erasure-era generics had to retrofit onto invariant raw types, and use-site variance lets *each call* pick the variance it needs without the type's author anticipating every usage. The trade-off: more verbose call sites and the cognitive load of PECS at every API boundary — but maximum flexibility and full backward compatibility.

## Senior Pitfalls

- Don't reach for `? extends` on a collection you need to **mutate** — you'll fight the compiler; you want plain `T` or `? super`.
- Mixing arrays and generics (`List<String>[]`) is forbidden precisely because array covariance would defeat generic invariance.
- Two `?` in the same signature are *different* unknowns — capture won't unify them.

## See Also

- [[generics-and-erasure]]
- [[type-system-and-conversions]]
- [[01-programming-foundations/paradigms/generics-and-polymorphism|Generics & Polymorphism]]
- [[01-programming-foundations/paradigms/type-systems|Type Systems]]
