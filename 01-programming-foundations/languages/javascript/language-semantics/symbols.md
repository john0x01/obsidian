# Symbols

A Symbol is a primitive whose only purpose is to be a guaranteed-unique, immutable identity. Two symbols are never equal, even with the same description, which makes them ideal as collision-proof property keys and as the hooks by which the language exposes its own extensibility points. They are the closest JavaScript comes to a built-in "private name" and the backbone of its meta-programming surface.

## Symbols As Unique Keys

`Symbol(desc)` mints a fresh value every call; the `desc` is purely a debugging label with no semantic weight:

```js
Symbol("id") === Symbol("id");   // false — distinct identities
typeof Symbol();                 // "symbol" (it is a primitive)
```

Used as a property key, a symbol cannot collide with any string key or any other symbol. This is the safe way to attach metadata to objects you don't own, or to add a "field" that user code won't accidentally overwrite or enumerate. Symbol-keyed properties are deliberately invisible to `for...in`, `Object.keys`, and `JSON.stringify`; they surface only via `Object.getOwnPropertySymbols` or `Reflect.ownKeys`. This semi-hidden quality is why they read as "soft private" — discoverable by reflection, but never by accident. (True privacy is class `#fields`, which are not symbols at all and are entirely unreflectable.)

## Well-Known Symbols — Protocol Hooks

The spec reserves a set of symbols on the `Symbol` constructor that the engine itself looks up to customize built-in behavior. Implementing one wires your object into a language protocol:

- `Symbol.iterator` — makes an object iterable for `for...of`, spread, destructuring (sync iteration protocol).
- `Symbol.asyncIterator` — the async counterpart, consumed by `for await...of`.
- `Symbol.hasInstance` — overrides `instanceof`; `obj instanceof C` calls `C[Symbol.hasInstance](obj)`. Lets you build duck-typed type checks without a real prototype link.
- `Symbol.toPrimitive` — intercepts coercion to primitive, receiving the hint (`"number"`/`"string"`/`"default"`); takes precedence over `valueOf`/`toString`.
- `Symbol.toStringTag` — customizes the tag in `Object.prototype.toString`, so `Object.prototype.toString.call(x)` returns `"[object YourTag]"`.
- `Symbol.species` — lets a subclass control which constructor methods like `Array.prototype.map` use to build derived instances.
- `Symbol.match`, `Symbol.replace`, `Symbol.search`, `Symbol.split` — let custom objects act like RegExp for `String.prototype` methods.

```js
class Temp {
  constructor(c){ this.c = c; }
  [Symbol.toPrimitive](hint){
    return hint === "string" ? `${this.c}°C` : this.c;
  }
}
const t = new Temp(20);
`${t}`;        // "20°C"  (string hint)
t + 5;         // 25      (default/number hint)
```

The design insight: rather than reserving magic string method names (which could collide with user data or be enumerated), the spec uses unforgeable symbol keys. Your object opts into engine behavior by defining a key only the engine knows to look for — a clean, collision-free extensibility mechanism.

## The Global Registry — Symbol.for / Symbol.keyFor

`Symbol()` produces a value unique to one place in one realm. When you need the *same* symbol shared across modules, files, or even realms (iframes, workers, the same Node process), use the global registry:

```js
Symbol.for("app.id") === Symbol.for("app.id");   // true — interned by string key
Symbol.keyFor(Symbol.for("app.id"));             // "app.id"  (reverse lookup)
Symbol.keyFor(Symbol("x"));                       // undefined — not registered
```

`Symbol.for(key)` returns the existing registry symbol for that string or creates and registers one. This is the cross-cutting coordination tool — e.g. a library tagging objects with `Symbol.for("react.element")` so different copies/versions can still recognize the brand. Registry symbols are *not* garbage-collected and live for the process lifetime, so reserve them for genuinely global, stable identities and namespace the keys (`"vendor.feature"`) to avoid registry collisions.

## Use-Cases And Meta-Programming

- **Branding / type tags**: stamp objects with a private symbol to identify "instances" without `instanceof` realm fragility.
- **Hidden metadata**: caches, internal flags, or back-references that must not appear in serialization or enumeration.
- **Enum-like constants**: each symbol is a distinct, self-documenting sentinel that cannot be confused with a string of the same text.
- **Protocol participation**: making custom collections iterable, awaitable-ish, or coercible by defining well-known symbols — the basis of how libraries integrate with native syntax.

## Senior Pitfalls And Mental Model

- Symbols are not enumerated or serialized — convenient for hidden state, but a trap if you expect `JSON.stringify`/`{...spread}` to preserve them (spread *does* copy own enumerable symbol keys; `JSON` drops them entirely).
- They cannot be auto-coerced to a string: `"" + Symbol()` throws; you must call `String(sym)` or `sym.description`/`.toString()` explicitly.
- Registry symbols never get collected — don't `Symbol.for` per-request data.
- They are *not* a security boundary; `getOwnPropertySymbols` exposes them. Use `#fields` for real privacy.

Mental model: a `Symbol()` is a fresh, anonymous key nobody else can guess; well-known symbols are the *named* keys the engine itself guesses, turning your object into a participant in built-in syntax; and `Symbol.for` is the one escape hatch when uniqueness must be shared by name rather than by identity.

## See Also

- [[iterators-and-generators]]
- [[property-descriptors-and-proxies]]
- [[type-system-and-coercion]]
- [[prototypes-and-inheritance]]
- [[01-programming-foundations/paradigms/type-systems|Type Systems]]
