# Pattern Matching and Switch

Pattern matching lets a single construct test a value's shape, conditionally bind its parts to variables, and (in `switch`) require that all shapes are handled. It collapses the old test-cast-extract dance into one declarative expression and, combined with records and sealed types, turns the compiler into a checker for data-oriented logic.

## `instanceof` Patterns and Flow Scoping

The type pattern in `instanceof` tests the type and, on success, binds a fresh variable already cast to that type:

```java
if (obj instanceof String s && !s.isBlank()) {
    use(s);            // s is in scope here, definitely a String
}
```

The subtlety is **flow scoping**: the binding `s` is in scope precisely where the compiler can prove the pattern matched. After `&&` it is in scope (short-circuit guarantees the match held); after `||` it would not be. It even works through negation and early returns: `if (!(obj instanceof String s)) return;` leaves `s` in scope *after* the `if`, because control only continues when the match succeeded. Flow scoping is not lexical — it follows the boolean dataflow.

## Switch Expressions

The arrow form replaces fall-through (and the bug factory of forgotten `break`) with isolated branches, and `switch` becomes an *expression* yielding a value:

```java
int days = switch (month) {
    case JAN, MAR, MAY, JUL, AUG, OCT, DEC -> 31;
    case APR, JUN, SEP, NOV -> 30;
    case FEB -> 28;
};
```

Each arm is a single expression, or a `{ ... }` block that produces its result with `yield`. As an expression, a `switch` must be **exhaustive** — every possible input must be covered — so it always produces a value. The colon form still exists with fall-through for backward compatibility, but the arrow form is the default for new code.

## Type and Record Patterns in `switch`

A switch can match on the runtime type of its selector, binding as it goes, and **record patterns** deconstruct a record into its components recursively:

```java
String describe(Shape s) {
    return switch (s) {
        case Circle(double r)            -> "circle r=" + r;
        case Rectangle(var w, var h)     -> "rect " + w + "x" + h;
    };
}
```

`Circle(double r)` matches a `Circle` and binds its component to `r` in one step. Patterns nest: `case Line(Point(var x1, var y1), Point p2)` destructures two levels deep. `var` infers the component type. This is structural decomposition — the shape of the pattern mirrors the shape of the data.

## Guards, Exhaustiveness, and `null`

A **guarded pattern** adds a boolean condition with `when`; the arm matches only if the pattern matches *and* the guard holds:

```java
case Integer i when i > 0 -> "positive";
case Integer i           -> "non-positive";
```

Order matters: a more specific guarded case must precede the catch-all, and the compiler rejects a case that is *dominated* by an earlier one (unreachable).

**Exhaustiveness against sealed hierarchies** is the centerpiece. When the selector is a sealed type and every permitted subtype has a case, the switch is exhaustive *without* a `default`. Omitting `default` is the point: add a new permitted subtype later and the switch fails to compile until you handle it — a compiler-enforced TODO list. (The compiler may still insert a synthetic fallthrough that throws if an unexpected subtype appears at runtime, e.g. due to separate compilation.)

`null` handling changed deliberately. A traditional `switch` throws `NullPointerException` on a null selector. A pattern switch still does *unless* you add an explicit `case null` (which may combine as `case null, default ->`). This keeps null handling visible and opt-in rather than a silent surprise.

## Primitive-Type Patterns (Preview)

Patterns matching primitive types directly — e.g. `case int i` in `switch`, or `instanceof int` — remain a **preview** feature in Java 25, not finalized. They extend exhaustiveness and binding to primitives and primitive-aware conversions. Treat them as not-yet-stable; don't rely on them in production builds.

## The Shift to Data-Oriented Programming

Pattern matching reframes how you process data. Instead of asking objects to act on themselves via dynamic dispatch, you take transparent data (records), model the alternatives as a closed set (sealed types), and write the logic as an exhaustive switch *outside* the data. The compiler guarantees you covered every case and bound every field correctly. The senior mindset shift: polymorphism distributes behavior *into* types and hides data; pattern matching pulls behavior *out* and exposes data — and you choose per problem which is the better fit.

## See Also

- [[records-and-sealed-types]]
- [[type-system-and-conversions]]
- [[enums-and-annotations]]
- [[01-programming-foundations/paradigms/pattern-matching|Pattern Matching]]
- [[01-programming-foundations/paradigms/functional-programming|Functional Programming]]
