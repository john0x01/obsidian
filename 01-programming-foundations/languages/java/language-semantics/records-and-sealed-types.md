# Records and Sealed Types

Records and sealed types are the two halves of Java's answer to algebraic data types. Records give you transparent, shallowly-immutable *product* types (a value with named components); sealed hierarchies give you closed *sum* types (a value that is exactly one of a known set of alternatives). Together they let the compiler reason about your data completely enough to enable exhaustive pattern matching and a "data-oriented" style that runs orthogonal to classical OO encapsulation.

## Records: Transparent Data Carriers

A record is declared by its *state description* — its component list — and the compiler derives everything else from it:

```java
record Point(int x, int y) {}
```

This generates: a private final field per component, a public accessor named exactly like the component (`x()`, not `getX()`), a canonical constructor, and `equals`/`hashCode`/`toString` derived structurally from the components. The class is implicitly `final`, and the components are `final`. "Transparent" is the key word: the API contract is that the record's state *is* its components, and the generated members must honor that — `equals` returns true iff all components are equal, and an accessor must return the corresponding component's value.

Records are *shallowly* immutable. The reference fields are final, but a `record Holder(List<String> items)` can still hand out a mutable list. If you want deep immutability you defensively copy in a **compact constructor**:

```java
record Range(int lo, int hi) {
    Range {                        // compact form: no parameter list, no field assignment
        if (lo > hi) throw new IllegalArgumentException();
        // params are the canonical params; final assignment is implicit
    }
}
```

The compact constructor runs validation/normalization on the canonical parameters; the field assignments happen automatically afterward. You may also write a full canonical constructor, or additional constructors that delegate via `this(...)` to the canonical one.

Restrictions encode the transparency contract: a record cannot `extend` any class (it already extends `java.lang.Record`), cannot declare additional instance fields, and cannot be abstract. It *can* implement interfaces, declare static members, and override the generated methods (rarely wise). It can be `local` to a method and can be generic.

## Sealed Types: Closed Hierarchies

A sealed class or interface restricts which types may directly extend or implement it via a `permits` clause:

```java
sealed interface Shape permits Circle, Rectangle {}
record Circle(double r) implements Shape {}
record Rectangle(double w, double h) implements Shape {}
```

Every permitted subtype must itself declare its openness: `final` (no further extension), `sealed` (continues the closed hierarchy with its own `permits`), or `non-sealed` (deliberately reopens the hierarchy to arbitrary subclasses). The `permits` clause may be omitted when all permitted subtypes are declared in the same source file (or, with modules, the same module/package). The crucial property: the compiler knows the *complete* set of direct subtypes at compile time.

## Why They Belong Together: ADTs and Exhaustiveness

A sealed interface is a sum: a `Shape` is a `Circle` *or* a `Rectangle`, and nothing else. A record is a product: a `Circle` is a radius *and* nothing more. This is exactly an algebraic data type. The payoff is **exhaustiveness checking** in `switch`: because the compiler knows the closed set, a switch over `Shape` that handles `Circle` and `Rectangle` needs no `default`, and *adding* a new permitted subtype turns every such switch into a compile error until you handle it. That converts "I forgot a case" from a production bug into a build failure — the single biggest ergonomic win of the design.

## Design Intent vs Traditional OO

Classical OO says: hide the data, expose behavior, and dispatch polymorphically so adding a new subtype requires touching no existing code (the open/closed principle). Records and sealed types deliberately invert this. They make data public and the type set closed, optimizing for the *other* axis: adding a new *operation* over a fixed shape of data is cheap and checked, while adding a new subtype is intentionally expensive (it breaks every switch). This is the classic "expression problem" trade-off. Pick records + sealed when the data shapes are stable and the operations grow — message types, AST nodes, command results, API DTOs. Pick polymorphic hierarchies when the operations are stable and you expect open-ended new implementations (plugins, strategies). The point is not that one replaces the other; it is that Java now lets you choose the axis of extensibility per problem instead of forcing the OO default everywhere.

## See Also

- [[pattern-matching-and-switch]]
- [[equality-and-contracts]]
- [[type-system-and-conversions]]
- [[01-programming-foundations/paradigms/inheritance-vs-composition|Inheritance vs Composition]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
