# Exceptions and Error Handling

Exceptions are Java's mechanism for separating the normal control flow from the handling of abnormal conditions, propagating failures up the stack until something is positioned to deal with them. Java's distinguishing choice — *checked* exceptions enforced by the compiler — is one of the most debated decisions in the language, and understanding the machinery (and the critique) is essential to using it well.

## The `Throwable` Hierarchy

Everything thrown extends `java.lang.Throwable`, which splits into two branches with very different intent:

- **`Error`** — serious conditions a program normally cannot recover from: `OutOfMemoryError`, `StackOverflowError`, `LinkageError`. You should not catch these (except at the outermost monitoring boundary).
- **`Exception`** — recoverable conditions. Its subtree splits again: `RuntimeException` and its descendants are **unchecked**; everything else under `Exception` is **checked**.

```
Throwable
├── Error              (unchecked — don't catch)
└── Exception
    ├── RuntimeException (unchecked — programming errors)
    └── (other)         (checked — recoverable, compiler-enforced)
```

The dividing line is *programmer error vs environmental failure*. `NullPointerException`, `IllegalArgumentException`, `IndexOutOfBoundsException` signal bugs — you fix the code, you don't catch them. Checked exceptions (`IOException`, `SQLException`) signal external conditions the caller might legitimately handle.

## The Checked-vs-Unchecked Design Decision

A checked exception is part of a method's *signature*: the caller must either `catch` it or declare `throws`. The compiler enforces a recovery contract — you cannot silently ignore the possibility of failure. The intent is admirable: make failure modes visible and force a conscious decision at every layer.

The critique, voiced by many senior practitioners, has two prongs. First, **boilerplate**: checked exceptions push noise through every signature on the call path. Second, the **leaky-abstraction** problem: a method's `throws SQLException` leaks its implementation (it uses a database) into its API; change the implementation and the signature — and every caller — must change. In practice this pressure produces the worst outcome of all: the empty `catch` block that swallows the exception to satisfy the compiler. Because of this, much modern Java (and most frameworks — Spring wraps `SQLException` into unchecked `DataAccessException`) leans toward unchecked exceptions, reserving checked ones for genuinely recoverable, caller-actionable conditions.

## Try-With-Resources and Suppressed Exceptions

Any object implementing `AutoCloseable` can be managed by try-with-resources, which guarantees `close()` is called in reverse order of acquisition, on both normal and exceptional exit:

```java
try (var in = Files.newInputStream(p);
     var out = Files.newOutputStream(q)) {
    in.transferTo(out);
}   // out.close() then in.close(), always
```

This solves a notorious pre-Java-7 trap: if both the `try` body *and* a manual `finally`-`close()` threw, the close exception would mask the original. Try-with-resources fixes this with **suppressed exceptions** — the body's exception propagates, and any exception from `close()` is attached to it and retrievable via `getSuppressed()`. Nothing is lost.

## Pitfalls: `finally` Return and Cause Chaining

A `return` (or `throw`) inside `finally` **silently discards** any exception or return value in progress:

```java
try { throw new IOException(); }
finally { return 0; }   // swallows the IOException entirely — never do this
```

The `finally` block's abrupt completion overrides the `try`'s. Never `return` or `throw` from `finally`; treat it as cleanup-only.

When you wrap one exception in another, always pass the original as the **cause** (`new ServiceException("load failed", e)`). The cause chain preserves the root stack trace through layers; dropping it destroys the diagnostic trail. "Catch low-level, throw domain-level *with cause*" is the disciplined pattern.

## The Cost of Stack Traces

Constructing a `Throwable` calls `fillInStackTrace()`, which walks the call stack — genuinely expensive, and the dominant cost of exceptions. This is why exceptions for *control flow* (e.g. a parse loop that throws on every miss) can be pathologically slow. For legitimately hot, expected failures you can override `fillInStackTrace()` to a no-op (or use the protected constructor that disables stack-trace writability) to make a stackless exception. But the right default is: don't use exceptions for ordinary control flow at all.

## Wrap vs Propagate, and the Philosophical Debate

Propagate when the current layer cannot do anything useful; wrap (with cause) when you're crossing an abstraction boundary and the lower-level type would leak. Catch only what you can actually handle.

The bigger debate: Java is essentially alone in enforcing checked exceptions. C# rejected them deliberately; Kotlin has none (all exceptions are unchecked, and `throws` is informational only). The argument against is that compiler-enforced recovery doesn't scale across deep stacks and degrades into noise or swallowing; the argument for is that it makes failure modes a visible, typed part of the contract — much as `Optional` makes absence visible rather than implicit `null`. The modern consensus: prefer unchecked for programming errors and infrastructure failures, reserve checked for narrow, recoverable, caller-actionable cases, and never let the compiler bully you into swallowing.

## See Also

- [[nullability-and-optional]]
- [[design-philosophy]]
- [[io-streams-and-readers]]
- [[java-vs-javascript]]
- [[01-programming-foundations/paradigms/functional-programming|Functional Programming]]
