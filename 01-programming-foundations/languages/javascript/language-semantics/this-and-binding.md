# This And Binding

`this` is not a variable and not lexically scoped (except in arrows). It is an implicit parameter bound *at the call site* according to how a function is invoked, not where it is defined. Internalizing "look at the call, not the definition" resolves the overwhelming majority of `this` confusion.

## The Four Call-Site Rules

When a normal function runs, the engine sets `this` by checking, in priority order:

1. **`new` binding** — `new Fn()` creates a fresh object, sets it as `this`, runs the body, and returns it (unless the body returns its own object). Highest precedence.
2. **Explicit binding** — `fn.call(obj, ...)`, `fn.apply(obj, argsArray)`, or a function permanently `fn.bind(obj)`. `bind` returns a new function whose `this` is locked; even `new` on a bound function ignores the bound `this` (but keeps bound args).
3. **Implicit binding** — `obj.fn()` sets `this = obj`. The receiver is whatever is left of the final dot at the moment of the call.
4. **Default binding** — a bare `fn()` call. In strict mode `this` is `undefined`; in sloppy mode it falls back to the global object (`window`/`globalThis`).

```
            new?  ──►  the new object
            call/apply/bind? ──►  the explicit object
            obj.fn()? ──►  obj
            else ──►  undefined (strict) / global (sloppy)
```

## Arrow Functions: Lexical This

Arrows have no `this` binding of their own at all — nor `arguments`, `super`, or `new.target`. They close over `this` from the enclosing lexical scope at definition time, exactly like any other captured variable. None of the four rules apply: `arrow.call(obj)` cannot change its `this`, and arrows cannot be used as constructors (`new arrow` throws). This is precisely why `arr.map(x => this.transform(x))` works inside a method while `arr.map(function(x){ return this.transform(x); })` loses `this` — the inner classic function gets default binding.

## The Lost-This Pitfall

Implicit binding is fragile because it depends on the *call expression*, not on where the reference lives. Detaching a method drops its receiver:

```js
const o = { name: "A", greet(){ return this.name; } };
const g = o.greet;     // reference only — no call yet
g();                   // this is undefined → throws / "undefined"

setTimeout(o.greet, 0);          // same loss — passed as a bare callback
el.addEventListener("click", o.greet);  // receiver becomes the element
```

Fixes: `o.greet.bind(o)`, an arrow wrapper `() => o.greet()`, or defining the handler as a class field arrow `greet = () => {...}` so it captures the instance lexically. Class methods are also affected — that is why React class components historically bound handlers in the constructor.

## This In Modules And Strict Mode

ES modules execute in strict mode automatically, and crucially the module top-level `this` is `undefined` (not the global object). In CommonJS, top-level `this` is `module.exports`. Inside any strict-mode function, default binding yields `undefined` rather than silently coercing to global — this prevents the accidental global-pollution bug where a forgotten `new` writes properties onto `window`. Standalone primitives passed via `call`/`apply` are *not* boxed in strict mode (`this` stays the raw primitive), whereas sloppy mode wraps them.

## Mental Model And Senior Notes

Think of `this` as a hidden first argument the call site supplies. Reading the body tells you nothing; you must trace the invocation. A few consolidations worth holding:

- **Definition site decides for arrows; call site decides for everything else.** That single sentence is the whole topic.
- `bind` is a one-way ratchet — you cannot re-bind a bound function, and chained `bind`s keep the first.
- Getters/setters and `Symbol.toPrimitive` hooks run with `this` set to the object they were accessed on, which is how computed properties read sibling state.
- `this` interacts with the prototype chain only as a *receiver*: an inherited method runs with `this` pointing at the calling instance, so `this.#field` and overridden methods resolve dynamically — this is dynamic dispatch.
- Tagged template tags, generator/async function bodies, and event handlers all still obey these rules; async functions preserve `this` across `await` because the body is one lexical function.

The pragmatic doctrine: prefer arrows for callbacks where you want enclosing `this`, prefer real methods (with the object as receiver) where you want dynamic dispatch, and never rely on default binding — it is almost always an accident.

## See Also

- [[scopes-and-closures]]
- [[prototypes-and-inheritance]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JavaScript Runtime]]
- [[01-programming-foundations/paradigms/object-oriented-programming|Object-Oriented Programming]]
- [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]
