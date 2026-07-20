# Enums and Annotations

Enums and annotations are Java's two mechanisms for attaching *fixed, compiler-known information* to code: enums model a closed set of named instances, annotations attach metadata to declarations. Both look like syntactic sugar but compile to real, inspectable structures — understanding what the compiler actually generates is what separates correct use from cargo-culting.

## Enum Internals

An `enum` is sugar for a `final` class that extends `java.lang.Enum<E>`, with each constant compiled to a `public static final` field holding a single eagerly-constructed instance. Because the constructor is effectively private and instances are created once in a static initializer, enum constants are guaranteed singletons — even across serialization (the runtime resolves by name) and reflection (which is blocked from instantiating them). This is why "an enum with one constant" is the canonical bulletproof singleton.

The compiler synthesizes two static methods not inherited from `Enum`: `values()` returns a fresh array of the constants in declaration order, and `valueOf(String)` looks one up by name (throwing `IllegalArgumentException` on miss). `name()` and `ordinal()` come from `Enum` itself.

Constants can carry state and behavior. Enums can have fields, constructors, and methods; each constant can supply constructor arguments and even override methods with a **constant-specific class body**:

```java
enum Op {
    PLUS  { int apply(int a, int b) { return a + b; } },
    TIMES { int apply(int a, int b) { return a * b; } };
    abstract int apply(int a, int b);
}
```

Each such constant compiles to an anonymous subclass of the enum. This gives polymorphism over a closed set without a separate hierarchy.

`EnumSet` and `EnumMap` exploit the dense, known ordinals. `EnumSet` is typically backed by a single `long` bit vector (or an array of longs for large enums) — set membership is a bit test, union/intersection are bitwise ops. `EnumMap` is backed by a flat array indexed by ordinal. Both are dramatically faster and lighter than `HashSet`/`HashMap` for enum keys and should be the default choice.

### The Ordinal-Fragility Pitfall

`ordinal()` returns the declaration position, and *reordering or inserting constants silently changes it*. Never persist or transmit `ordinal()`; never use it as a database column, serialized form, or protocol value. Use `name()` (or an explicit, stable code field you assign yourself). The compiler will not warn you — this is a classic latent data-corruption bug.

## Annotations as Metadata

An annotation is a special interface (it implicitly extends `java.lang.annotation.Annotation`) whose elements look like methods with optional defaults. Annotations attach metadata to declarations (and, since type annotations, to type uses). They do nothing on their own; some *processor* must read them.

Two meta-annotations govern an annotation's reach. **`@Retention`** selects its lifespan:

- `SOURCE` — discarded after compilation (e.g. `@Override`, lint hints); visible only to the compiler and source tools.
- `CLASS` — kept in the `.class` file but not loaded at runtime (the default); usable by bytecode tools.
- `RUNTIME` — retained and readable via reflection at runtime.

**`@Target`** restricts where it may appear (`TYPE`, `METHOD`, `FIELD`, `PARAMETER`, `TYPE_USE`, etc.). `@Repeatable` allows multiple instances on one element by designating a container annotation. `@Inherited` makes a class-level annotation visible on subclasses through reflection.

## Two Consumption Models: APT vs Reflection

The retention policy dictates *who* can read the annotation, and the two consumers have opposite cost profiles.

- **Compile-time annotation processing (APT)**, via `javax.annotation.processing` and `RoundEnvironment`, runs *inside javac*. Processors inspect the program model (elements/types) and can generate new source files in additional rounds. Cost is paid once at build time; there is zero runtime overhead and errors surface during compilation. This is how Lombok, Dagger, MapStruct, and `record`-adjacent tooling work — they read `SOURCE`/`CLASS` annotations and emit code.
- **Runtime reflection** reads `RUNTIME`-retained annotations via `Class`/`Method`/`Field`'s `getAnnotation`. It is flexible but pays a per-access cost and defers errors to startup or first use.

Frameworks mix both. JPA's `@Entity`/`@Column` and Spring's `@Component`/`@Autowired`/`@Transactional` are `RUNTIME` annotations scanned reflectively at startup to build metadata, wire beans, and generate proxies. The modern trend (Spring AOT, Micronaut, Quarkus) is to shift this scanning to *build time* to slash startup cost and enable native images — pushing reflection-era work back toward the APT model.

## See Also

- [[reflection-and-method-handles]]
- [[type-system-and-conversions]]
- [[records-and-sealed-types]]
- [[pattern-matching-and-switch]]
- [[01-programming-foundations/paradigms/aspect-oriented-programming|Aspect-Oriented Programming]]
