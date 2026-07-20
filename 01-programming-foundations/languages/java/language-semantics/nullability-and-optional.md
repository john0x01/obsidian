# Nullability and Optional

`null` is the universal **bottom value** for reference types — the absence of an object, assignable to any reference, holding no fields and answering no methods. Tony Hoare, who introduced it in 1965, later called it his "**billion-dollar mistake**." Java inherited it, and dereferencing it produces the language's most infamous runtime failure: `NullPointerException`. This note is about living with that mistake and the tools — chiefly `Optional` — that contain it.

## The Mechanics of `null` and NPE

Every reference variable can hold `null`; primitives cannot. An NPE is thrown the instant you treat `null` as if it pointed to an object: method call, field access, array indexing, unboxing, or `synchronized` on it. Crucially, NPEs are *unchecked* (`RuntimeException`) — the compiler never forces you to handle them, because the type system has no notion of "this reference cannot be null."

Before JDK 14, an NPE in `a.b.c.d()` told you only the line, not *which* dereference was null. **Helpful NullPointerExceptions** (JEP 358, JDK 14) changed that: the JVM now reconstructs the exact expression, e.g. `"Cannot invoke \"C.d()\" because \"a.b.c\" is null"`. It's on by default in modern JDKs and is one of the highest-leverage debugging improvements in the language.

## `Optional`: A Return Type, Not a Field

`Optional<T>` is a container that is either *present* (holds a non-null `T`) or *empty*. Its purpose is to make "no result" **explicit in the type signature** of a method that may legitimately return nothing — `findById`, `stream().max()`, etc. It is a *vocabulary type for return values*, and using it elsewhere is the most common misuse.

**Do not** use `Optional` for:

- **Fields** — it adds an allocation and an extra indirection per object, and `Optional` is **not `Serializable`** (a deliberate design choice signaling it's not meant to be a stored data carrier).
- **Method parameters** — callers must wrap arguments; an overload or `@Nullable` is cleaner.
- **Collection elements** — an empty collection already encodes "nothing."

## Composing Optionals — and the `orElse` Trap

The point of `Optional` is fluent composition without null checks:

```java
String name = repo.findUser(id)        // Optional<User>
    .map(User::profile)                // Optional<Profile>
    .flatMap(Profile::displayName)     // flatMap when the fn returns Optional
    .orElseGet(() -> defaultName());   // lazy fallback
```

- `map` transforms a present value; `flatMap` does the same but *flattens* when your function itself returns `Optional` (avoids `Optional<Optional<X>>`).
- **`orElse(x)` vs `orElseGet(supplier)`**: `orElse` is **eagerly evaluated — always**, even when the value is present. So `optional.orElse(expensiveCall())` runs `expensiveCall()` every time, present or not. Use `orElseGet` for any non-trivial or side-effecting fallback so it's computed *only* when empty. This is the single most common `Optional` performance bug.
- Avoid `optional.isPresent()` followed by `optional.get()` — that's just a null check with extra steps. Prefer `map`/`ifPresent`/`orElse*`. Never call `get()` without proving presence.

## `Objects.requireNonNull` — Fail Fast

For arguments and invariants that must not be null, validate at the boundary:

```java
this.name = Objects.requireNonNull(name, "name");
```

This throws an NPE *immediately* with a clear message at the point of the bad input, rather than letting a `null` propagate and explode three frames deeper. Constructors and public methods should guard their inputs this way — it converts a mystery NPE into a precise, documented contract violation.

## Why No Built-In Non-Null Types

Java has **no language-level non-null type** — you cannot declare `String!` to mean "never null" the way Kotlin can. Nullability lives only in **external annotations** (`@Nullable` / `@NonNull` from JSR-305, JetBrains, JSpecify, etc.), which tooling (IDEs, NullAway, the Checker Framework) enforces *statically* but the JVM ignores entirely. Retrofitting genuine non-null types onto a language where every reference has always been nullable is a backward-compatibility nightmare — hence the annotation ecosystem instead of a built-in.

## Future Direction: Valhalla

Project **Valhalla** introduces *null-restricted* types for value classes (where a flattened value object has no natural null), pointing toward first-class nullability tracking. It is **not shipped** — a roadmap item, not a current tool.

## Contrast: JS `undefined` vs `null`

JavaScript has *two* bottoms: `undefined` (never assigned / missing property) and `null` (intentional absence), with `==` coercing them equal but `===` distinguishing them. Java collapses both notions into a single `null`, which is simpler but loses the "never set" vs "explicitly empty" distinction. The trade: JS gives you two ambiguous holes; Java gives you one — and `Optional` to express "deliberately empty" without overloading `null`.

## See Also

- [[equality-and-contracts]]
- [[exceptions-and-error-handling]]
- [[type-system-and-conversions]]
- [[stream-api]]
- [[01-programming-foundations/languages/javascript/language-semantics/type-system-and-coercion|JS Type System & Coercion]]
