# Module Systems

JavaScript shipped without a module system. Every system in use today — CommonJS, AMD, UMD, ESM — is an answer to one question the language refused to answer for two decades: *how do independent files share code without colliding in a single global namespace?* Understanding them as successive answers to that question, each constrained by the runtime it targeted, is the mental model that makes the trade-offs obvious.

## The Era of Globals

Before any module convention, a `<script>` tag's top-level declarations leaked straight onto `window`. Load order was the dependency graph, maintained by hand. Two libraries declaring `$` clobbered each other; minifiers could not safely rename anything reachable from the global object. The entire history of modules is the progressive *encapsulation* of this shared scope and the progressive *formalization* of those implicit load-order dependencies into explicit declarations.

## IIFE and the Module Pattern

The first encapsulation primitive was the function. An immediately-invoked function expression creates a private scope; you expose a curated surface by returning an object (or assigning to a single global). Dependencies are passed in as arguments — making them explicit and minifiable.

```js
var MyLib = (function ($, _) {
  var privateState = 0;            // not reachable from outside
  function publicApi() { /* ... */ }
  return { publicApi: publicApi }; // the export surface
})(jQuery, lodash);
```

This is the conceptual ancestor of every later system: a closure for privacy, an explicit dependency list, a single returned namespace. UMD (Universal Module Definition) is just a header on this pattern that *detects* which loader is present (`module.exports`? `define`? neither?) and wires exports to the right place — a portability shim, not a system of its own.

## CommonJS — Synchronous, Dynamic, Server-First

CommonJS was designed for the server (Node), where files live on a local disk and synchronous reads are cheap. Its model is procedural: `require(x)` is an ordinary function call that *executes* the target module top-to-bottom and returns whatever its `module.exports` object currently holds. `module.exports` is a mutable value; `exports` is just a starting alias to it.

The defining property is that the module structure is **dynamic**. Because `require` is a function, you can call it conditionally, compute the specifier at runtime, or reassign `module.exports` mid-file. The set of imports and exports is not knowable without running the code. This flexibility is exactly what makes static tooling (tree-shaking, reliable cross-module inlining) hard.

## AMD — Asynchronous, for the Browser

The browser cannot block on a disk read; it must fetch over the network. AMD (RequireJS) answered with `define(id, [deps], factory)`: declare dependencies up front so the loader can fetch them asynchronously, then invoke your factory once they resolve. It solved async loading but at the cost of verbose, callback-shaped boilerplate. AMD and CJS coexisted by audience (browser vs server), and UMD bridged libraries across both.

## ESM — Static Structure as a Language Feature

ES Modules (ES2015) made modules part of the *language*, not a library convention, and made a deliberate, consequential choice: **import/export are static, syntactic forms, not runtime calls.** They may only appear at the top level; specifiers are string literals. This means the complete import/export graph of a program is knowable by parsing alone, before any code runs.

That single decision unlocks the properties ESM exists for:

- **Static analysis & tree-shaking** — unused exports are provably dead and can be dropped.
- **Live bindings** — importers see an *exported binding*, not a copied value; later mutations in the exporter are visible to importers. (CJS copies the value present at `require` time.)
- **Asynchronous loading** — the engine can resolve and fetch the whole graph ahead of evaluation, suiting the network.
- **Cyclic dependencies** handled at the binding level rather than via half-initialized `exports` objects.

`import()` (dynamic import) deliberately re-introduces runtime, conditional loading as an *async* operation returning a promise — the escape hatch for code-splitting and lazy loading, without sacrificing the static guarantees of the top-level form.

## CJS vs ESM — Semantic Differences That Bite

- **Sync vs async**: `require` is synchronous and returns immediately; `import` participates in an async graph. You cannot `require` an ESM module's evaluation synchronously into CJS.
- **Value copy vs live binding**: `const { x } = require('m')` snapshots `x`; `import { x } from 'm'` tracks it.
- **`this`**: top-level `this` is `module.exports` in CJS, `undefined` in ESM.
- **Strict mode**: always on in ESM; opt-in in CJS.
- **Hoisting**: ESM imports are hoisted and resolved before evaluation; CJS `require` runs in place.
- **No `__dirname`/`require` in ESM** (use `import.meta.url`); no top-level `await` in CJS.

The philosophy split: CJS optimizes for *runtime flexibility* (a module is a program that produces an object); ESM optimizes for *ahead-of-time knowability* (a module is a declared graph). Most interop pain — named-export detection, dual-package hazards, the inability to synchronously consume async modules — flows directly from trying to bridge those two philosophies.

## See Also
- [[esm-internals]]
- [[module-resolution]]
- [[async-await]]
- [[08-quality-and-operations/devops-and-infrastructure/build-systems|Build Systems]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
