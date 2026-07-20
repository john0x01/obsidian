# The Java Module System (JPMS)

The Java Platform Module System (JPMS), delivered by Project Jigsaw in Java 9, adds a unit of program structure *above* the package: the module. A module is a named, self-describing collection of packages with an explicit dependency list and an explicit public surface. It exists to solve two problems the classpath never could — the JDK had grown into an unbreakable monolith, and Java had no way to enforce real encapsulation across JAR boundaries.

## What `module-info.java` Declares

A module is defined by a `module-info.java` at the root of its sources, compiled to a `module-info.class`. The directives are the entire contract:

```java
module com.acme.orders {
    requires com.acme.core;            // I depend on this module
    requires transitive java.sql;      // and anyone who requires me gets java.sql too
    exports com.acme.orders.api;       // this package is public to all readers
    exports com.acme.orders.spi to com.acme.billing;  // qualified: only to billing
    opens com.acme.orders.model;       // deep reflection (e.g. frameworks) allowed
    provides com.acme.orders.api.Codec with com.acme.orders.internal.JsonCodec;
    uses com.acme.orders.api.Codec;    // I consume this service via ServiceLoader
}
```

Two distinctions matter. `requires transitive` makes a dependency *implied* — if your API returns a `java.sql.Connection`, you re-export `java.sql` so callers compile without re-declaring it. And `opens` is separate from `exports`: exporting grants compile-time and normal-reflection access to public members; opening grants *deep reflective* access (including to private members via `setAccessible`). The split exists because frameworks like Jackson or Hibernate need to crack open your model classes, but you don't want that capability granted to everyone for everything.

## Readability and Accessibility

Access in JPMS is the conjunction of two rules. **Readability**: module A can reference module B only if A `requires` B (the module graph defines a reads-edge). **Accessibility**: a public type in B is reachable from A only if B `exports` its package. A `public` class in a non-exported package is now genuinely unreachable from outside — `public` is no longer a global guarantee. This is the heart of *strong encapsulation*: `sun.misc.*` and other JDK internals that were always technically `public` became inaccessible, finally ending the decades of code reaching into undocumented internals.

## The Module Graph and Reliable Configuration

At startup the runtime resolves the *module graph* from the root modules: it transitively walks `requires` edges, and fails fast if any module is missing, if two modules export the same package (a split package — illegal), or if a cycle in `requires` is detected. This is **reliable configuration**: the old classpath silently tolerated missing JARs until the offending class was first loaded, then threw `NoClassDefFoundError` mid-flight, possibly in production. JPMS converts that into a deterministic launch-time check. `java.base` is the implicit root every module reads; you never declare it.

## Services via ServiceLoader

`provides ... with` and `uses` formalize the service-provider pattern. A consumer declares `uses Codec` and calls `ServiceLoader.load(Codec.class)`; the runtime discovers every module that `provides Codec with SomeImpl` and instantiates them. Unlike the old `META-INF/services` file-scanning, this is part of the validated module graph — the binding is declared, type-checked, and visible to tooling and to jlink, which can prune unreachable providers.

## Why Jigsaw Existed — The Design Philosophy

Two goals drove it. First, **de-monolithing the JDK**: splitting `rt.jar` into ~ modules (`java.base`, `java.sql`, `java.desktop`, …) so the platform could be reasoned about and, crucially, *subset* — enabling jlink to build runtimes containing only what an app needs. Second, **scalable strong encapsulation**: giving libraries a way to hide internals that the compiler *and* the runtime enforce, so an API's published surface is a real boundary rather than a documentation convention. The cost is real friction — every reflective framework had to learn `opens`, and migrating a classpath app is non-trivial — which is why adoption of *application* modules remains partial even as the *platform* modularization is universal and irreversible.

## See Also

- [[class-loading]]
- [[class-loading]]
- [[jlink-and-runtime-images]]
- [[reflection-and-method-handles]]
- [[design-philosophy]]
- [[01-programming-foundations/languages/javascript/modules/module-systems|JS Module Systems]]
