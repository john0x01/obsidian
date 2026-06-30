# Scopes And Closures

Scope is the set of bindings reachable from a point in source code; a closure is a function that keeps that scope alive after the enclosing function has returned. Both are explained by one runtime structure — the chain of *Environment Records* — and understanding it turns closures from mysterious to mechanical.

## Lexical Scoping And The Scope Chain

JavaScript is lexically (statically) scoped: a binding's visibility is determined by *where it is written*, not by the dynamic call stack. At parse time the engine establishes a nesting of scopes; at run time each scope is realized as an **Environment Record** — a structure holding that scope's bindings plus an `[[OuterEnv]]` pointer to the enclosing record. Resolving an identifier walks this chain inward-out: current record, then outer, then outer, up to the global record. The chain mirrors the source nesting exactly, which is why you can predict resolution by reading the code, not by tracing who called whom.

```
inner fn env ──outer──► outer fn env ──outer──► module env ──outer──► global env
```

## How Closures Keep Variables On The Heap

Normally a function's environment record is reclaimed when the call returns. A closure forms when an inner function still references the outer record: the inner function value carries an internal `[[Environment]]` slot pointing at the record where it was created. Because that record is reachable, the garbage collector cannot free it — so the *variables*, not copies of them, survive on the heap.

```js
function counter(){
  let n = 0;                 // lives in counter's env record
  return () => ++n;          // closes over that record
}
const c = counter();
c(); c();                    // 1, 2 — same n, mutated, still alive
```

The captured `n` is shared, live state — not a snapshot. Two functions closing over the same record see each other's writes; this is how private state and module patterns work. Engines optimize by capturing only the variables actually referenced (escape analysis), but semantically the whole accessible record persists.

## var/let/const, Hoisting, And TDZ

The binding *region* differs by keyword and explains hoisting:

- `var` is function-scoped (ignores blocks) and is hoisted to the top of its function, initialized to `undefined`. Reading it before the assignment yields `undefined`, not an error.
- `let`/`const` are block-scoped. They are also "hoisted" to the top of the block in the sense that the block reserves the name, but they sit in the **Temporal Dead Zone** — uninitialized — until execution reaches the declaration. Touching them in the TDZ throws `ReferenceError`. This converts a silent `undefined` bug into a loud one.
- Function *declarations* hoist entirely (name + body), so they are callable above their text. Function *expressions* follow their assignment keyword's rules.

`const` freezes the *binding*, not the value: `const a = []` forbids reassigning `a` but permits `a.push(1)`.

## The Classic Loop Closure

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i));  // 3 3 3
for (let i = 0; i < 3; i++) setTimeout(() => console.log(i));  // 0 1 2
```

With `var` there is one function-scoped `i`; all three closures share it and read its final value. With `let`, the spec creates a *fresh binding per iteration* and copies the value forward, so each closure captures its own `i`. This is the single most cited demonstration of "closures capture bindings, not values," and the `let` fix is exactly per-iteration environment records.

## Memory Implications And Patterns

Closures are a deliberate leak: they pin scopes. Practical consequences and uses:

- **Leaks**: a long-lived closure (event listener, cache, timer) that captures a large object keeps it alive indefinitely. The fix is to null references or remove listeners. A subtle trap: all closures over the same record share its lifetime, so one small closure can retain a big sibling variable it doesn't even use (older engines), though modern V8's escape analysis usually trims unreferenced captures.
- **Encapsulation / module pattern**: an IIFE returning an object whose methods close over private locals — pre-`#`-fields privacy.
- **Factories & partial application**: returning specialized functions that bake in captured arguments (`const add5 = makeAdder(5)`).
- **Memoization**: a cache held in the enclosing scope, persisting across calls.
- **Currying & function composition** rely on each returned function retaining earlier arguments.

## Mental Model And Senior Notes

Picture every function value as carrying a backpack — a pointer to the environment where it was born. Calls create new records linked to that backpack; resolution always follows the *lexical* outer chain, never the caller. Hold these consolidations: scope is decided at authoring time; closures keep bindings (mutable, shared) rather than values; `let`/const + TDZ exist to make hoisting bugs throw instead of returning `undefined`; and the same mechanism that gives you private state is also your most common memory leak. Treat captured references as ownership — know what each long-lived function is keeping alive.

## See Also

- [[this-and-binding]]
- [[iterators-and-generators]]
- [[01-programming-foundations/paradigms/closures|Closures]]
- [[01-programming-foundations/languages/javascript/execution-model/garbage-collector|Garbage Collector]]
- [[01-programming-foundations/paradigms/higher-order-functions|Higher-Order Functions]]
