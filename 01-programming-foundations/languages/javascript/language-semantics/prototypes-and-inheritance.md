# Prototypes And Inheritance

JavaScript has no classes at the runtime level — only objects linked to other objects. Inheritance is *delegation*: when a property is missing on an object, the engine follows a hidden link to another object and retries. Every higher-level construct (`class`, `new`, `extends`) is sugar over this one mechanism, and seeing through the sugar is what separates fluent use from cargo-culting.

## The `[[Prototype]]` Slot vs `.prototype`

These are different things and conflating them is the classic confusion:

- `[[Prototype]]` is an internal slot every object has, pointing to its prototype (or `null`). You read/write it via `Object.getPrototypeOf` / `Object.setPrototypeOf`, or the legacy `__proto__` accessor.
- `.prototype` is an ordinary property that exists **only on functions**. It holds the object that will become the `[[Prototype]]` of instances created by calling that function with `new`.

```
function Dog(){}
const d = new Dog();
Object.getPrototypeOf(d) === Dog.prototype   // true
Dog.prototype.constructor === Dog            // true (back-reference)
```

So `Dog.prototype` is *not* Dog's own prototype (`Dog`'s is `Function.prototype`); it is the template for Dog's instances.

## Property Lookup, Shadowing, And The Chain

Reading `obj.x` walks the chain: own property → `[[Prototype]]` → its `[[Prototype]]` → … → `Object.prototype` → `null`, returning the first match (or `undefined`). The chain terminates at `null`, which is why `Object.create(null)` makes a truly bare dictionary with no inherited `toString`/`hasOwnProperty`.

*Writing* does not walk the chain the same way. `obj.x = 1` creates or updates an **own** property, shadowing any inherited `x` — unless an inherited `x` is an accessor (setter) or a non-writable data property, in which case assignment is redirected to the setter or silently ignored (throws in strict mode). This asymmetry is the source of many bugs: mutating an inherited array/object property mutates the shared prototype copy, while assigning a primitive shadows it locally.

## Object.create — Delegation In The Raw

`Object.create(proto, descriptors?)` makes a new object whose `[[Prototype]]` is exactly `proto`. This is prototypal inheritance with no constructors involved — the purest expression of "objects linked to objects." It is also how `class extends` wires up the chain under the hood.

## Constructor Functions Vs `class`

Before `class`, inheritance was hand-assembled:

```js
function Animal(name){ this.name = name; }
Animal.prototype.speak = function(){ return `${this.name} makes a sound`; };

function Dog(name){ Animal.call(this, name); }      // borrow constructor
Dog.prototype = Object.create(Animal.prototype);    // link chains
Dog.prototype.constructor = Dog;                     // restore back-ref
```

`class` automates exactly this. `extends` sets up the dual chain (instance side *and* the static side, `Dog.__proto__ === Animal`), `super()` performs the `Animal.call(this)` step, and methods land on `.prototype` rather than on each instance. Critical differences from functions, though: class bodies run in strict mode, class declarations are not hoisted (TDZ applies), constructors throw if called without `new`, and methods are non-enumerable. So `class` is *mostly* syntactic sugar but adds real semantic guardrails.

## Class Fields And Private #

Instance fields (`count = 0`) are assigned in the constructor on each instance, not on the prototype — equivalent to writing `this.count = 0`. Private fields (`#secret`) are categorically different from a closure or a `_convention`: they are enforced by the engine, accessible only lexically within the class body, invisible to `Object.keys`, reflection, and `Proxy` traps, and a `#field in obj` check is the safe brand-test for "is this a genuine instance." They are stored via a separate internal mechanism, not as string-keyed properties, so they never appear in the prototype chain at all.

## Delegation Vs Classical Inheritance

The philosophical pivot: classical OOP *copies* structure from class to instance at construction; prototypes *delegate* lookups at access time to live, shared objects. Practical implications:

- Shared methods exist once on the prototype — memory-efficient, but monkey-patching the prototype affects all existing instances retroactively (live link, not a snapshot).
- Deep chains cost lookup time and couple types rigidly; this is the runtime echo of the "favor composition over inheritance" wisdom.
- Delegation enables patterns classical languages can't easily express: borrowing methods across unrelated objects, mixins via `Object.assign(Cls.prototype, mixin)`, and runtime re-parenting (though `setPrototypeOf` is a deopt — engines pessimize objects whose prototype mutates).

The senior mental model: a `class` is a factory plus a shared method bag, and an instance is a thin own-property record delegating everything else up a chain. Reach for composition and plain objects by default; use `class`/`extends` when a genuine is-a hierarchy and `instanceof` checks earn their keep.

## See Also

- [[this-and-binding]]
- [[property-descriptors-and-proxies]]
- [[type-system-and-coercion]]
- [[01-programming-foundations/paradigms/object-oriented-programming|Object-Oriented Programming]]
- [[01-programming-foundations/paradigms/inheritance-vs-composition|Inheritance vs Composition]]
