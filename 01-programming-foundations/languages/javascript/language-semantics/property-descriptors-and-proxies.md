# Property Descriptors And Proxies

Every property in JavaScript is more than a key-value pair: it carries metadata (a *descriptor*) that controls whether it can be written, enumerated, or reconfigured, and it may be backed by functions rather than a stored value. Proxies take this further, letting you intercept the fundamental operations themselves. Together they are the meta-programming substrate that reactivity libraries, ORMs, and validation layers are built on.

## Property Descriptors

A property is defined by one of two descriptor shapes, sharing two attributes:

- `enumerable` — appears in `for...in`, `Object.keys`, and spread.
- `configurable` — may be deleted or have its descriptor changed (the master lock).

A **data descriptor** adds `value` and `writable`. An **accessor descriptor** adds `get` and `set` functions instead. The two are mutually exclusive. Defaults differ by creation path, and this catches people: literal assignment (`obj.x = 1`) produces `{writable:true, enumerable:true, configurable:true}`, but `Object.defineProperty` defaults every omitted attribute to `false`. So `Object.defineProperty(o, "x", {value:1})` creates a read-only, non-enumerable, locked property — a frequent surprise.

```js
Object.getOwnPropertyDescriptor([].constructor.prototype, "length");
// arrays: length is {writable:true, enumerable:false, configurable:false}
```

## Getters, Setters, And defineProperty

Accessor properties run code on read/write while looking like plain fields. They power computed/derived values, lazy initialization (a getter that replaces itself with a data property on first access), and validation on assignment. `this` inside an accessor is the object it was accessed on (respecting the prototype chain), so getters defined on a prototype read instance state. `Object.defineProperty` / `defineProperties` is the low-level tool to set all this precisely; class `get`/`set` syntax is sugar that installs non-enumerable accessors on the prototype.

## Object Integrity Levels

Layered locks, weakest to strongest:

- `preventExtensions` — no new properties; existing ones still writable/deletable.
- `seal` = `preventExtensions` + all properties `configurable:false` (can't delete or redefine, can still write values).
- `freeze` = `seal` + all data properties `writable:false` (shallow immutability).

All are shallow — `Object.freeze(obj)` does not freeze nested objects. `Object.isFrozen`/`isSealed` query the level.

## Proxy And Reflect

A `Proxy` wraps a *target* with a *handler* whose methods are **traps** intercepting internal operations: `get`, `set`, `has` (the `in` operator), `deleteProperty`, `ownKeys`, `getOwnPropertyDescriptor`, `defineProperty`, `apply` (calling), `construct` (`new`), `getPrototypeOf`, and more. Unhandled traps pass straight through to the target.

```js
const p = new Proxy(target, {
  get(t, key, receiver){ return Reflect.get(t, key, receiver); },
  set(t, key, val, receiver){ /* validate/notify */ return Reflect.set(t, key, val, receiver); }
});
```

`Reflect` exposes those same internal operations as plain functions, mirroring the trap signatures one-to-one. Two reasons it matters inside traps: it returns a useful value/boolean instead of throwing (so `set` can report success correctly), and it forwards `receiver` so that getters/setters on the prototype chain see the proxy as `this`, preserving correct dispatch.

## Meta-Programming And Reactivity

This is the engine behind modern reactivity. Vue 3's reactivity proxies an object so that `get` records which effect is reading a property (dependency tracking) and `set` re-runs the effects that depended on it (trigger). Unlike Vue 2's `Object.defineProperty` approach — which had to walk and convert every property up front and could not detect property *addition* or array index writes — a `Proxy` intercepts the operation generically, including new keys and deletions, lazily and on the whole object. MobX, immer (copy-on-write via proxies), and ORMs use the same hooks for change tracking, lazy loading, and virtual properties.

## Invariants — The Hard Limits

Proxies cannot lie about *non-configurable* or *non-writable* reality on the target. The engine enforces invariants: if a target property is non-configurable and non-writable, a `get` trap must return its actual value; `getPrototypeOf` must agree with a non-extensible target's real prototype; `ownKeys` must include all non-configurable own keys and cannot fabricate keys for a non-extensible target. Violating an invariant throws a `TypeError`. This keeps proxies from breaking the structural guarantees other code relies on. Private `#fields` are also invisible to traps — a method using `this.#x` on a proxy throws unless the trap unwraps to the raw target.

## Senior Pitfalls And Mental Model

- Descriptors are the truth; assignment is a convenience that fabricates a permissive descriptor. When something "won't change," inspect its descriptor.
- `freeze` is shallow — reach for deep-freeze utilities or structural immutability when you need real guarantees.
- Proxies add per-operation overhead and break `===` identity with the target; never proxy a hot path naively, and never compare proxy to target by reference.
- Always route trap logic through `Reflect` with `receiver` to keep prototype/`this` semantics intact.

The model: descriptors describe a single property's powers; integrity levels lock objects; proxies + Reflect virtualize the object's *behavior* itself — meta-programming is just intercepting the operations the language usually performs silently.

## See Also

- [[symbols]]
- [[prototypes-and-inheritance]]
- [[iterators-and-generators]]
- [[01-programming-foundations/paradigms/reactive-programming|Reactive Programming]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
