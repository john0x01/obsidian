# CommonJS Internals

CommonJS is Node's original module system: a synchronous, file-based loader where `require()` is an ordinary function call that returns a value, not a parsed-ahead declaration. Understanding it as *plain function calls executing top-to-bottom* — rather than as static syntax — explains every one of its quirks, including the cache, circular dependencies, and the `module.exports`/`exports` confusion that trips up engineers years into their careers.

## The Module Wrapper

A `.js` file is never executed as raw script. Before evaluation, Node wraps its source in a function:

```js
(function (exports, require, module, __filename, __dirname) {
  // your file's code
});
```

This is the single fact that explains most of CommonJS. The wrapper is why top-level `var`/`const` declarations are module-scoped (they are function-locals, not globals), why `__dirname` and `__filename` exist without being globals (they are parameters), and why `require`, `module`, and `exports` are available without importing anything. Node compiles this wrapped string with V8, then invokes the resulting function with the freshly-constructed `module` object, a per-module bound `require`, and the path variables. There is no magic syntax — `require` is `Module.prototype.require`, and `module.exports` is just a property on an object.

## `module.exports` vs `exports`

`exports` starts as a reference to the same object as `module.exports`:

```text
module.exports ──► { }  ◄── exports
```

Mutating that shared object works through either name (`exports.foo = 1` is identical to `module.exports.foo = 1`). But **reassigning** `exports` only rebinds the local parameter; the module still returns whatever `module.exports` points at. This is why `exports = function(){}` silently exports nothing, while `module.exports = function(){}` works. The senior mental model: `require()` returns `module.exports`, full stop. `exports` is a convenience alias that you lose the moment you reassign it.

## The Module Cache

`require.cache` is keyed by **resolved absolute path**. The first `require('./x')` runs the wrapper and stores the resulting `module` object; every subsequent `require` of the same resolved path returns the cached `module.exports` *without re-executing the file*. Consequences:

- Modules are effectively singletons per path. Top-level side effects (opening a connection, registering handlers) run exactly once.
- Two different specifiers resolving to the same file share state; the same package installed at two paths in `node_modules` does **not**.
- A package duplicated in the dependency tree (e.g. two `instanceof`-incompatible copies of a class) is a classic "it's the same class but `instanceof` fails" bug — different cache keys, different module instances.
- Deleting `require.cache[id]` forces a reload; test frameworks and hot-reload tools exploit this, but it leaks the old instances if anything still holds references.

## Synchronous Resolution and Loading

`require` is fully synchronous. Resolution (mapping a specifier to a file path) and loading (reading + compiling + executing) both block the event loop. This is deliberate: it gives strict, deterministic ordering and lets `require` *return a value*. It is also why large CJS trees inflate cold-start time — every `require` is a blocking `stat`/`open`/`read` plus compile. The synchrony is precisely what ESM gave up to gain its async, statically-analyzable graph.

## Circular Dependencies

Because loading is synchronous and execution is eager, a cycle resolves to a **partially-populated** export object. When A requires B, and B requires A *while A is still executing*, B receives whatever A had assigned to `module.exports` up to that point — not the finished module:

```js
// a.js
exports.done = false;
const b = require('./b'); // runs b fully now
exports.done = true;

// b.js
const a = require('./a'); // a is mid-execution
console.log(a.done);      // false — A hasn't finished yet
```

The cache entry for A exists before A finishes (Node inserts it pre-execution to break infinite recursion), so B gets the in-progress object. The pitfall: code that reads cyclic imports *at module top level* sees stale/undefined values, whereas deferring access into a function (called after both modules finish) sees the complete exports. Cycles are not errors in CJS — they fail quietly with `undefined`, which is far more dangerous than a hard error. The philosophy: CJS prioritizes "always make progress" over correctness guarantees, leaving cycle hazards as the user's problem.

## Senior Pitfalls

- Mutating `module.exports` after an async tick — too late; consumers already captured the reference.
- Assuming `require` is free; it is I/O + compile on the hot path.
- Relying on cache for cross-realm singletons (workers, child processes have separate caches).

## See Also

- [[esm-in-node]]
- [[module-resolution]]
- [[01-programming-foundations/languages/javascript/modules/module-systems|Module Systems]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
