# Type System And Coercion

JavaScript is dynamically and weakly typed: values carry types, bindings do not, and the engine will silently convert between types according to deterministic spec algorithms rather than rejecting mismatches. Mastering coercion means knowing those algorithms cold, because they explain every "weird" result you will ever hit.

## Primitives Vs Objects

There are seven primitive types — `undefined`, `null`, `boolean`, `number`, `bigint`, `string`, `symbol` — plus `object` (which subsumes functions and arrays). Primitives are immutable, compared by value, and stored "by copy"; objects are mutable, compared by reference identity, and live on the heap. The deep distinction is *identity*: two strings `"ab"` are indistinguishable and `===`-equal, while `{}` and `{}` are forever distinct. This is why immutability is cheap for primitives and structural sharing matters for objects.

## ToPrimitive — The Root Algorithm

Most coercion funnels through the abstract operation `ToPrimitive(input, hint)`. When an object must become a primitive it runs an ordered method list determined by `hint`:

- `hint === "string"` → try `toString()`, then `valueOf()`.
- `hint === "number"` (the default for arithmetic, comparisons) → try `valueOf()`, then `toString()`.
- A `Symbol.toPrimitive` method, if present, overrides everything and is called with the hint.

Whichever method returns a primitive first wins; if neither does, a `TypeError` is thrown. This single ordering explains `[] + {}` → `"[object Object]"` (both stringify with the default+"number" path falling through to `toString`) and why `{} + []` can differ depending on parsing context.

## ToNumber And ToString

`ToNumber` parses strings (trimming whitespace, accepting hex/binary/octal/exponent/`Infinity`), maps `""`→`0`, `true`→`1`, `false`/`null`→`0`, `undefined`→`NaN`, and throws for `symbol`/`bigint`. `ToString` does the inverse for printing. Operators choose: binary `+` is overloaded — if *either* operand becomes a string after `ToPrimitive`, it concatenates; otherwise it adds numerically. All other arithmetic operators (`-`, `*`, `/`, `%`) force `ToNumber`, which is why `"5" - 1 === 4` but `"5" + 1 === "51"`.

## == Vs === And The Coercion Ladder

`===` (Strict Equality) compares type then value, no conversion. `==` (Abstract/Loose Equality) applies a fixed cascade:

```
null == undefined          → true (and ONLY each other)
number == string           → string ToNumber'd, retry
boolean == anything        → boolean ToNumber'd first (so true==1)
object == primitive        → object ToPrimitive'd, retry
NaN == anything            → false
```

Consequences worth memorizing: `0 == ""` (both →0), `0 == "0"`, `"" != "0"` (string-vs-string, no coercion), `null == 0` is **false** (the null/undefined rule short-circuits before numeric coercion), and `[] == ![]` is `true` (`![]`→`false`→`0`, `[]`→`""`→`0`). The senior stance: use `===` always; reserve `== null` as the idiomatic nullish check that catches both `null` and `undefined`.

## Truthiness

Boolean coercion has exactly eight falsy values: `false`, `0`, `-0`, `0n`, `""`, `null`, `undefined`, `NaN`. Everything else — including `"0"`, `"false"`, `[]`, `{}`, and all functions — is truthy. `if`, `&&`, `||`, ternaries, and `!` all run `ToBoolean`. `??` is different: it tests *only* nullish (`null`/`undefined`), so `0 ?? 5 === 0` whereas `0 || 5 === 5`.

## Boxing And Wrapper Objects

Accessing a property on a primitive (`"hi".length`) transparently wraps it in a temporary `String`/`Number`/`Boolean`/`Symbol`/`BigInt` object, reads through its prototype, then discards the wrapper. This is why you can call methods on primitives but cannot durably attach properties to them (`x = "a"; x.foo = 1; x.foo` → `undefined`). Avoid `new Number(5)` explicitly — the wrapper is a truthy object, breaking `==` and `typeof` expectations.

## NaN, -0, And Numeric Edge Cases

`NaN` is the only value not equal to itself; test with `Number.isNaN` or `Object.is`. `-0` and `+0` are `===`-equal and `==`-equal but distinguishable via `Object.is(-0, 0) === false` or `1/x` (`-Infinity` vs `Infinity`). `Object.is` implements SameValue semantics — like `===` but treating `NaN===NaN` and `-0!==0`. Numbers are IEEE-754 doubles, hence `0.1 + 0.2 !== 0.3`.

## typeof And instanceof

`typeof` is the only operator that never throws on undeclared identifiers (returns `"undefined"`). Its quirks are historical: `typeof null === "object"` (a frozen v1 bug), and functions report `"function"` while all other objects report `"object"`. `instanceof` walks the prototype chain checking for `Constructor.prototype`, and is customizable via `Symbol.hasInstance`; it is unreliable across realms (iframes/workers) where each has its own `Array.prototype`.

## Philosophy And Pitfalls

Weak typing was a deliberate "be liberal in what you accept" design for a browser scripting toy. The trade-off: convenience and resilience at the cost of silent failure modes that surface far from their cause. The senior mental model is to treat coercion as *explicit by intent* — convert at boundaries (`Number(input)`, `String(x)`, `Boolean(v)`), never rely on implicit `+`/`==` ladders in logic, and let TypeScript or runtime guards re-introduce the static discipline the language withholds.

## See Also

- [[prototypes-and-inheritance]]
- [[symbols]]
- [[scopes-and-closures]]
- [[01-programming-foundations/paradigms/type-systems|Type Systems]]
- [[01-programming-foundations/paradigms/immutability|Immutability]]
