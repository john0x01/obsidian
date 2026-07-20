# Lambdas and Functional Interfaces

A lambda is a compact anonymous implementation of a *functional interface* — an interface with exactly one abstract method (a SAM, Single Abstract Method). Lambdas exist to let Java pass behavior as data without the ceremony of anonymous inner classes; the subtle part is that they are wired up at runtime through `invokedynamic`, not compiled to inner-class files. That indirection is what explains their performance profile, their `this` semantics, and how they differ from JavaScript's first-class functions.

## Functional Interfaces and Target Typing

A functional interface has exactly one abstract method. `default`, `static`, and `private` methods don't count, and neither do redeclared `Object` methods (`equals`, `hashCode`, `toString`). The optional `@FunctionalInterface` annotation asks the compiler to *enforce* the SAM rule so a later edit can't silently break lambda compatibility. The standard toolkit lives in `java.util.function` (`Function`, `Predicate`, `Consumer`, `Supplier`, `UnaryOperator`, …), joined by older SAMs like `Runnable`, `Callable`, and `Comparator`.

Crucially, a lambda has **no standalone type**. It acquires one from context — its *target type*, which must be a functional interface. The same `x -> x` is a `Function` in one position and a `UnaryOperator` in another. This target typing is why a lambda can't be assigned to a bare `var`, and why overloads taking different functional interfaces can become ambiguous and need a cast to disambiguate.

## Method References

Method references are sugar for a lambda that only forwards. Four shapes: static (`Integer::parseInt`), bound instance (`System.out::println` — receiver already fixed), unbound instance (`String::length` — the receiver becomes the *first* parameter), and constructor (`ArrayList::new`). They obey the same target-typing rules; the compiler picks the overload that matches the target SAM's shape.

## How Lambdas Actually Compile

Lambdas are **not** anonymous inner classes. `javac` desugars the body into a private synthetic method in the enclosing class (named like `lambda$process$0`) and, at the use site, emits a single `invokedynamic` instruction whose bootstrap method is `LambdaMetafactory.metafactory`.

```text
use site:  invokedynamic  ──bootstrap──►  LambdaMetafactory
                                              │ spins a hidden class
                                              ▼ implementing the SAM,
              linked CallSite  ◄──────────  delegating to lambda$…$0
```

On first execution the JVM invokes the bootstrap, which spins an implementation class at runtime (today a *hidden class*) that implements the functional interface and delegates to the synthetic method, then returns a linked `CallSite`. Every later hit uses that linked, JIT-inlinable site — effectively free. Two payoffs: the translation strategy is deferred to runtime rather than frozen in bytecode (a future JDK can change it without recompiling your code), and there is no separate `.class` file per lambda, so class-loading is lighter. A **non-capturing** lambda links to a cached singleton (`ConstantCallSite`); a **capturing** one produces a fresh instance per evaluation that carries the captured values.

## Capture and Effectively-Final Variables

A lambda may capture *effectively-final* locals — variables never reassigned after initialization (the `final` keyword becomes optional). Locals are captured **by value** (copied into the instance); fields are reached through a captured `this` reference, so field reads and writes stay live. The value-capture rule is deliberate: a lambda handed to another thread sees a stable snapshot with no data race on a mutating local, keeping the model simple and safe.

## Contrast with JavaScript Closures

The gap with JS is instructive. JavaScript functions are first-class *values* with their own type that close over the **live binding** — mutable and shared through the environment record (see [[01-programming-foundations/languages/javascript/language-semantics/scopes-and-closures|Closures]]). Java lambdas are nominally-typed *instances of an interface* that close over a **copied value**. Hence the classic workaround for "mutate a captured counter": you can't reassign a captured `int`, so you mutate a captured object (`int[] c = {0}; run(() -> c[0]++);`) — in JS you would just close over a `let`.

On `this`, the mapping is clean: a Java lambda's `this` is **lexical** (the enclosing instance) — exactly like a JS arrow function — whereas an anonymous inner class's `this` is the inner instance, like a JS ordinary function. Migrating anon-class code to a lambda can therefore silently change what `this` means.

## Senior Pitfalls

- **Allocation.** Capturing lambdas allocate per evaluation; in hot loops prefer non-capturing forms (which are cached) or hoist the captured state.
- **Serialization.** Serializable lambdas take a slower, fragile path (`SerializedLambda`, a synthetic `$deserializeLambda$`); avoid unless required.
- **Checked exceptions.** Standard SAMs don't declare checked exceptions, forcing awkward try/catch wrapping inside the body — friction with Java's checked-exception model.
- **Overload ambiguity** from target typing needs an explicit cast.

The philosophy: lambdas let Java borrow functional ergonomics (behavior as data) while staying nominally typed and object-oriented underneath — every lambda is still an interface instance. Routing them through `invokedynamic` is the same instinct the JVM applies everywhere: add a layer of runtime indirection to preserve future freedom.

## See Also

- [[records-and-sealed-types]]
- [[generics-and-erasure]]
- [[nullability-and-optional]]
- [[01-programming-foundations/paradigms/functional-programming|Functional Programming]]
- [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]
- [[01-programming-foundations/languages/javascript/language-semantics/scopes-and-closures|Closures]]
