# Hidden Classes and Inline Caches

JavaScript objects are, semantically, dictionaries: keys can be added, deleted, and reordered at runtime, and property lookup is "search a map of strings to values." A naive engine would do a hash lookup on every `obj.x`, which is murderously slow for hot code. V8's two intertwined mechanisms — **hidden classes** (object *shapes*) and **inline caches** — recover near-static-language speed by betting that, at any given code location, objects almost always have the *same structure*. Understanding them is what separates "JS that the JIT can optimize" from "JS that secretly runs in the slow path."

## Hidden Classes (Shapes / Maps)

V8 gives every object a **hidden class** (internally a *Map*; "shape" is the cross-engine term) describing its layout: which properties it has and at what fixed offset each value sits. Objects with the *same* properties added *in the same order* share one hidden class. So the engine can store property values in a flat, array-like slot region and access `obj.x` as a constant offset — no string hashing.

Hidden classes are **immutable** and form a **transition tree**. Adding a property creates a *new* hidden class and a transition edge from the old one:

```
{}            -> C0
obj.x = 1     -> C0 --(add x)--> C1   (x at offset 0)
obj.y = 2     -> C1 --(add y)--> C2   (y at offset 1)

// another object built the same way reuses C0->C1->C2
```

Consequences that drive real performance:

- **Initialization order matters.** `{x, y}` and `{y, x}` end up in *different* hidden classes even though they have the same keys. Code that touches both must handle two shapes.
- **Construct objects consistently.** Initialize all fields in the constructor / object literal, in the same order, every time, so all instances converge on one shape. Conditionally adding fields later splinters shapes.
- **`delete` is poison.** Deleting a property typically forces the object out of its fast shape into **dictionary (slow) mode**, a real hash map. Prefer setting to `null`/`undefined`.
- **Sparse / huge-index arrays** also fall to dictionary mode; keep arrays dense and `PACKED`, ideally single-element-type (all Smis, all doubles, or all objects — V8 tracks **elements kinds** and transitions them one-way toward more general/slower).

## Inline Caches (ICs)

A hidden class makes the *offset* knowable; an **inline cache** is the per-call-site memory that *remembers* it. The first time a bytecode op like `LdaNamedProperty obj.x` runs, V8 records, in that op's feedback slot, "I saw hidden class C2; `x` lives at offset 0." On subsequent executions it checks "is `obj`'s hidden class still C2?" — a single pointer compare — and if so reads the offset directly, skipping the lookup entirely. The same machinery caches method targets at call sites (enabling inlining) and operator behavior.

ICs have a **state** per site, and this is the single most important performance lens:

- **Monomorphic** — one shape ever seen. Fastest: one guard, then direct access. This is the optimizer's happy path and the prerequisite for inlining.
- **Polymorphic** — a small set of shapes (V8's threshold is small, historically ~4). The IC stores a few (shape → offset) entries and linear-checks them. Still fast, but a step down, and harder to inline.
- **Megamorphic** — too many shapes; the IC gives up caching per-site and falls back to a generic (often global) lookup. Slow, and a hard blocker for optimization.

```
site obj.x:
  uninitialized -> monomorphic(C2)        // one shape, fastest
                -> polymorphic(C2,C5,C9)   // few shapes, linear probe
                -> megamorphic             // give up, generic hash lookup
```

A function written once but fed structurally different objects degrades its *own* call sites to polymorphic/megamorphic — even though the source never changed. The slowness is a property of the *data shapes flowing through*, not the syntax.

## Why Monomorphism Matters

The optimizing tiers (Maglev/TurboFan — see [[jit-compilation]]) consume IC feedback as their **type oracle**. A monomorphic IC tells the compiler "this is always shape C2," so it emits a fixed-offset load guarded by one shape check, and can *inline* a monomorphic method call into the caller. Polymorphic feedback forces guard chains; megamorphic feedback gives the compiler nothing to specialize on, so it emits the generic slow path and often can't inline at all. **Monomorphism is the contract between your data and the JIT.** Speculative optimization is only as good as the type information ICs collected — see [[deoptimization]] for what happens when that information turns out wrong.

## Practical Fast-JS Implications

- **Stable shapes**: same fields, same order, set in the constructor; avoid `delete`; avoid mutating shape after init (no "add a field on a hot path the 50th time").
- **Monomorphic call sites**: prefer functions that see one type of object; uniform arrays/collections over heterogeneous bags. A `map` over `{type:'a'}|{type:'b'}|...` unions can go polymorphic.
- **Don't mix elements kinds**: keep numeric arrays numeric; pushing an object into a packed-double array transitions it to a slower kind permanently.
- **Classes help**: ES `class` instances naturally share a shape, which is part of why class-based hot code tends to be monomorphic-friendly versus ad-hoc object literals built differently per branch.
- **Measure, don't guess**: `--trace-ic` and DevTools "Optimization" hints reveal which sites went mega/polymorphic. Premature shape-tuning of cold code is wasted effort — apply this only to profiled hot paths.

The mental model: V8 is *betting* your objects are statically shaped. Write code that honors the bet and you get C-like access; violate it (shape churn, `delete`, type-mixing) and you silently fall to dictionary lookups and lost inlining.

## See Also
- [[jit-compilation]] — how feedback drives speculative compilation
- [[deoptimization]] — when shape assumptions break
- [[parsing-and-bytecode]] — feedback vectors that store IC state
- [[v8-architecture]] — where shapes live in the heap model
- [[07-performance-engineering/cache-friendly-code|Cache-Friendly Code]] — data layout for speed
- [[03-computer-systems/computer-architecture/branch-prediction|Branch Prediction]] — speculate-and-guard at the hardware level
