# ESM In Node

ECMAScript Modules are Node's standards-based module system: a statically-analyzable, asynchronously-loaded graph with live bindings, layered onto a runtime that was born CommonJS. The defining tension is that ESM's loader is *async and multi-phase* while CJS's is *sync and single-phase*, and almost every interop rule and "missing global" exists to reconcile those two worlds.

## How Node Decides a File Is ESM

Node selects the module system before parsing, based on:

- File extension: `.mjs` is always ESM, `.cjs` is always CommonJS.
- Nearest `package.json` `"type"` field: `"module"` makes `.js` files ESM; `"commonjs"` (the default) makes them CJS.
- `--input-type=module` for `--eval`/STDIN.

This is a *static* decision — Node never sniffs source for `import`/`export`. Picking the wrong one yields `SyntaxError: Cannot use import statement outside a module` or the reverse.

## The Asynchronous, Three-Phase Loader

Unlike `require`, the ESM loader splits work into phases over the whole graph:

```text
1. Construct  — resolve every specifier, fetch & parse all sources,
                build the module record graph (no code runs yet)
2. Instantiate — wire up live bindings: each import name points at the
                 exporter's variable slot (not a copied value)
3. Evaluate   — run module bodies in post-order (deps before dependents)
```

Because phase 1 parses imports *statically* before any evaluation, the entire graph is known up front, enabling async fetching and (in tooling) tree-shaking. **Live bindings** are the key semantic difference from CJS: an imported name is a read-only *view* of the exporter's binding, so if the exporter later reassigns it, the importer sees the new value. CJS captures a value at `require` time; ESM tracks the variable.

## Top-Level Await

ESM module bodies may `await`. The evaluation phase becomes async: a module with TLA returns a promise, and any module importing it waits for that to settle before its own body runs. This makes ESM genuinely async end-to-end — and is one reason `require()` cannot natively load an arbitrary ESM file synchronously (you cannot block on a promise from sync code). Node 22 added `require(esm)` support for ESM graphs that contain *no* top-level await, throwing `ERR_REQUIRE_ASYNC_MODULE` otherwise.

## CJS ↔ ESM Interop

- **ESM importing CJS**: Node treats the CJS module's `module.exports` as the *default* export. Node also runs a static heuristic (the `cjs-module-lexer`) to detect named exports so `import { foo } from 'cjs-pkg'` often works — but only for names the lexer can see syntactically. Dynamically-assigned properties (`exports[name] = …` in a loop) are invisible; fall back to `import pkg from 'cjs-pkg'; const { foo } = pkg`.
- **CJS importing ESM**: historically impossible synchronously; use `await import()` (dynamic import returns a promise and works from CJS), or rely on `require(esm)` in Node 22 for TLA-free graphs.

## The Missing Globals

ESM has no `require`, `module`, `exports`, `__dirname`, or `__filename` — they were CJS wrapper parameters, and ESM has no wrapper. Replacements:

```js
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url); // for occasional CJS-only deps

import { fileURLToPath } from 'node:url';
const __filename = fileURLToPath(import.meta.url);
const __dirname = new URL('.', import.meta.url).pathname; // or path.dirname(__filename)
```

`import.meta.url` is the module's absolute `file:` URL — the canonical anchor for path math in ESM. Node 20.11+/21+ also exposes `import.meta.dirname` and `import.meta.filename`, which are the idiomatic Node 22 replacements.

## Dual Packages

A package that ships both formats uses conditional `"exports"`:

```json
{ "exports": { "import": "./index.mjs", "require": "./index.cjs" } }
```

The hazard is the **dual-package hazard**: if both copies load, you get two distinct module instances with separate state and incompatible `instanceof`. Mitigations: keep all stateful logic in one CJS core and make the ESM entry a thin re-export wrapper, or ship ESM-only. Philosophically, dual packaging is a transitional cost; the ecosystem is converging on ESM-first with CJS as a compatibility shim.

## Senior Pitfalls

- Expecting named imports from a CJS package whose exports are built dynamically — they won't be detected.
- Forgetting that ESM specifiers need file extensions (`./util.js`, not `./util`) unless an `"exports"`/`"imports"` map supplies them.
- Assuming `import` is hoisted side-effect-free: bodies still run once, in dependency order, and TLA can stall the whole graph.

## See Also

- [[commonjs-internals]]
- [[module-resolution]]
- [[01-programming-foundations/languages/javascript/modules/esm-internals|ESM Internals]]
- [[01-programming-foundations/languages/javascript/modules/module-systems|Module Systems]]
