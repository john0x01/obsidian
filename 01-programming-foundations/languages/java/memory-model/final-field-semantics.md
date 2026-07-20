# Final-Field Semantics

`final` fields receive a special, stronger guarantee from the Java Memory Model than ordinary fields. It is the reason immutable objects like `String` can be shared freely across threads with *no synchronization at all* and still always be observed fully initialized. Understanding this rule — and its one strict precondition — is what separates "I use immutability because it's tidy" from knowing *why* it is thread-safe.

## The Freeze Guarantee

For an ordinary field, a thread that obtains a reference to an object through a [[java-memory-model|data race]] may see the field as its default value (`0`/`null`) even after the constructor finished, because the field write and the reference publication can be reordered. `final` fields are exempt.

The JMM specifies a **freeze** action at the end of a constructor: when construction completes, all of the object's `final` fields are frozen. The guarantee is:

> If an object is **correctly constructed**, then any thread that reads a reference to that object — *even via a data race* — is guaranteed to see the correctly-initialized values of its `final` fields (and anything reachable through them at construction time).

So a properly built immutable object needs *no* `volatile`, no lock, no [[safe-publication|safe publication]] to be shared safely. The reference can be passed through a plain non-volatile field and readers still see consistent `final` state. This is unique — it is the only place the JMM gives a visibility guarantee without an explicit happens-before edge created by the *reader's* code.

```java
final class Point {
    final int x, y;
    Point(int x, int y) { this.x = x; this.y = y; }
    // After construction, x and y are frozen.
    // Any thread seeing a Point reference sees correct x, y — no sync needed.
}
```

## The Precondition: `this` Must Not Escape

The guarantee holds *only* if the object is **correctly constructed**, which has a precise meaning: **the `this` reference must not escape during construction.** If the constructor publishes `this` before it returns — even indirectly — another thread can grab the reference *before* the freeze, and then sees the fields mid-construction with no guarantee whatsoever.

Ways `this` escapes, all dangerous:

- Registering a listener/callback that captures `this` from inside the constructor.
- Starting a thread (`new Thread(...).start()`) inside the constructor that uses `this`.
- Storing `this` into a static field or shared collection before the constructor returns.
- A non-final method called from the constructor that an overriding subclass uses to leak `this`.

The mental model: the freeze is the fence. Anything that lets another thread observe the reference *before* the fence is past it loses the protection. This is why the "publish the object only after the constructor fully returns" discipline matters even for immutable types.

## Why Immutable Objects Are Safe To Race

Combine the two facts: (1) `final` fields are correctly visible to any thread that sees the reference, and (2) by definition an immutable object never changes after construction. Therefore there is *no write after construction to race on*, and the only write that matters — initialization — is covered by the freeze. An immutable object is thus **safe to share through arbitrary, even racy, channels**. This is the deep justification behind the [[01-programming-foundations/paradigms/immutability|immutability]] paradigm's thread-safety claim: immutability converts a concurrency problem into a non-problem.

This underwrites concrete platform guarantees. `String` is immutable with `final` fields, so interned or shared strings are always seen intact. **Records** are shallowly immutable — their components are implicitly `final` — so a record built from immutable components inherits the same race-safe publication property. Well-designed immutable value classes (all fields `final`, defensively copied mutable inputs, no `this` escape) get it for free.

## Senior Pitfalls

- A `final` field pointing at a **mutable** object: the *reference* is frozen and visible, but later mutations of the target object are ordinary writes with no guarantee. Freeze covers the field, not the deep object graph's post-construction changes.
- Assuming `final` implies the field is set safely under *all* circumstances — it is void if `this` escaped.
- Using reflection or `Unsafe`/`VarHandle` to mutate a `final` field after construction — this violates the model; the JIT may have constant-folded the field, so readers may never see the change.
- Believing `final` provides atomicity or mutual exclusion — it provides neither; it is purely an initialization-visibility guarantee.

## Design Philosophy

The freeze rule encodes a powerful idea: **construction is a publication boundary.** By making immutability the easy, default-safe path, the JMM nudges design toward value-like objects that need no locking. Make fields `final`, never leak `this`, and entire classes of visibility bugs become impossible — the cheapest concurrency strategy is the one where there is nothing to synchronize.

## See Also

- [[safe-publication]]
- [[java-memory-model]]
- [[records-and-sealed-types]]
- [[volatile-and-synchronized]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
