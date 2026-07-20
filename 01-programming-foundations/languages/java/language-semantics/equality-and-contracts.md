# Equality and Object Contracts

Java distinguishes **reference identity** (`==`, "same object") from **value equality** (`.equals`, "logically equal"). Because `Object.equals` defaults to identity, value semantics is something you *opt into* — and doing so correctly means honoring a set of contracts the entire collections framework silently depends on. Break them and `HashMap`, `HashSet`, and `TreeMap` quietly misbehave with no exception to warn you.

## The `equals` Contract

`equals` must define an **equivalence relation**:

- **Reflexive**: `x.equals(x)` is `true`.
- **Symmetric**: `x.equals(y) == y.equals(x)`.
- **Transitive**: `x.equals(y) && y.equals(z) ⇒ x.equals(z)`.
- **Consistent**: repeated calls return the same result if the objects don't change.
- **Non-null**: `x.equals(null)` is `false` (never throws).

## The `hashCode` Contract — and Why It's Load-Bearing

The binding rule: **equal objects must have equal hash codes.** (The converse is *not* required — unequal objects may collide.)

This exists because hash-based collections locate entries in two stages: first the **bucket** via `hashCode`, then linear `equals` *within* that bucket. If two equal objects hash differently, they land in different buckets and the collection never finds the match:

```
put(key)  -> bucket = hash(key) % n -> store
get(key2) -> bucket = hash(key2) % n -> wrong bucket -> miss
```

So overriding `equals` *without* `hashCode` makes objects that are `.equals` but invisible to each other in a `HashSet` — a classic, silent bug. Always override both together.

## What Breaks When You Violate the Contracts

- **Asymmetry under inheritance.** A subclass that adds a field to `equals` typically breaks symmetry: `sub.equals(super)` is `false` while `super.equals(sub)` is `true`. The standard fix is `getClass()` checks (rejects cross-class equality) or, better, **favor composition over inheritance** for value types — there is no clean way to extend an instantiable class and add a value component while preserving the contract. This is *why* records are `final`.
- **Mutable keys.** If a key's `equals`/`hashCode` depend on a mutable field and you mutate it *after* insertion, its stored bucket no longer matches its new hash. The entry becomes a ghost — present in memory, unreachable by `get`. **Keys should be effectively immutable.**
- **Identity vs value confusion.** Using `==` on `Integer`, `String`, or any value-bearing object is a latent bug (it happens to "work" only via interning/caching).

## `Comparable`, `compareTo`, and Consistency-With-Equals

`Comparable<T>` defines a **natural ordering** via `compareTo`, returning negative/zero/positive. It must be a *total order*: antisymmetric, transitive, and `sgn(x.compareTo(y)) == -sgn(y.compareTo(x))`.

The subtle rule: `compareTo` **should be consistent with `equals`** — `x.compareTo(y) == 0` *should* imply `x.equals(y)`. It's not strictly required, but **sorted collections use `compareTo`, not `equals`, to decide equality.** A `TreeSet` will treat two objects with `compareTo == 0` as duplicates even if `equals` says they differ — the canonical gotcha is `BigDecimal`, where `new BigDecimal("1.0").compareTo(new BigDecimal("1.00")) == 0` but `equals` is `false`, so the two behave differently in a `HashSet` versus a `TreeSet`.

`Comparator<T>` externalizes ordering when you need an alternative to (or replacement for) the natural one — composable via `comparing`, `thenComparing`, `reversed`. Prefer it over hand-rolled comparisons; the combinators are correct by construction and avoid integer-overflow bugs from naive `a - b` subtraction.

## How Records Generate These

A `record` auto-generates `equals`, `hashCode`, and `toString` from its components, by value, in declaration order. The generated `equals` is **component-wise** and respects the contract; `hashCode` derives from the same components. Because records are implicitly `final` and their components are `final`, they sidestep the inheritance-asymmetry and mutable-key traps by design — making them the right default for keys, map values, and DTOs. (Floating-point components use bitwise comparison, so `NaN` equals `NaN` and `0.0` differs from `-0.0` — a deliberate, contract-preserving choice.)

## Senior Mental Model

Identity equality answers "is this the *same* object?"; value equality answers "do these *mean* the same thing?". Most domain types want value semantics — reach for a record. Most framework-managed entities (JPA, identity caches) deliberately want identity or carefully scoped equality. The contracts aren't bureaucracy: they're the invariants that let hashing and sorting be O(1)/O(log n) instead of O(n) scans.

## See Also

- [[records-and-sealed-types]]
- [[nullability-and-optional]]
- [[type-system-and-conversions]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
- [[01-programming-foundations/paradigms/inheritance-vs-composition|Inheritance vs Composition]]
