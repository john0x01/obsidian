# ESM Internals

An ES module is not loaded in one step. The spec defines a three-phase lifecycle — **construction, instantiation (linking), evaluation** — and the separation is the whole point: it is what allows live bindings, deterministic cycle handling, and the static guarantees the format promises. Treat ESM not as "files that import each other" but as a graph the engine builds, wires, and only then runs.

## Phase 1 — Construction (Parse → Module Records)

Starting from an entry specifier, the loader resolves it to a URL, fetches the source, and **parses** it into a *Module Record* (specifically a Source Text Module Record). Parsing extracts the module's static shape without executing anything: its list of `import` requests, its declared exports, and an *Environment Record* (the lexical scope) that will hold the bindings.

Crucially, parsing reveals the imports, so the loader recurses: it resolves and fetches each dependency, building the full **module graph**. The loader maintains a **module map** keyed by resolved URL/key, so each module is fetched and constructed exactly once even when reached through many paths. The map is also the deduplication and memoization layer — a second `import` of the same key returns the same record, never a re-fetch.

```
construction:   parse entry → discover imports → fetch deps → recurse
                build module map { key → ModuleRecord }, edges = imports
```

## Phase 2 — Instantiation / Linking

Once the graph exists, the engine performs a depth-first **Link** pass over it. No user code runs in this phase. For each module it:

1. Creates the module's Environment Record and allocates a slot for every local binding and every export.
2. Resolves each import to the **exact binding** in the exporting module's environment (via `ResolveExport`, which follows re-exports and star-exports).
3. Connects importer slots to exporter slots so they reference the *same* underlying binding cell.

This is the mechanism behind **live bindings**: `import { count } from './m.js'` does not copy a value; the importer's `count` *is* a read-through reference to `m.js`'s `count` cell. When `m.js` later does `count++`, every importer observes the new value. Imports are also read-only views — you cannot assign to an imported binding. This is fundamentally different from CommonJS, where `const { count } = require('./m')` copies whatever value sat in `exports.count` at that instant; later mutation in the module is invisible.

Because linking resolves names before evaluation, an `import` of a binding that the target never exports is a **compile-time error**, surfaced before any code executes — unlike a stray `undefined` from a missing CJS property.

## Phase 3 — Evaluation

Finally the engine runs each module body, depth-first, post-order: dependencies evaluate before the modules that import them, and the module map guarantees each body runs **once**. Errors thrown during evaluation are cached on the record, so a re-import of a module that already threw rethrows the same error rather than re-running.

By evaluation time the binding cells already exist (from linking) but hold their *temporal-dead-zone* state until the relevant declaration executes — which is what makes circular dependencies tractable.

## Circular Dependencies

Suppose `a.js` imports from `b.js` and vice versa. The graph is built once (the map prevents infinite fetch). During post-order evaluation one module runs first; if it *reads* a binding from the still-unevaluated cyclic partner, it sees the binding cell in its TDZ. Two outcomes:

- If the read happens at module **top level** before the partner has assigned it → `ReferenceError` (accessing a `let`/function-export before init), or `undefined` for a `var`-style hoist.
- If the read is deferred inside a **function** that runs later (after both modules finish evaluating) → it works, because by call time the live binding is populated.

This is strictly better than CJS cycles, where you receive a *partially-populated `exports` object* and silently get `undefined` with no diagnostic. The senior takeaway: in a cycle, reference cyclic imports lazily (inside functions), never eagerly at top level.

## Top-Level Await

`await` at module top level makes a module **asynchronous**. The engine splits evaluation: an async module's evaluation returns a promise, and any module importing it must wait for that promise before its own body runs. TLA therefore propagates up the graph — importers of an async module become async themselves — and the engine still guarantees a deterministic, dependency-ordered evaluation. The cost is that TLA can *delay* the entire subgraph rooted above it; overuse serializes startup.

## Dynamic `import()`

`import(specifier)` is a runtime operation (an expression, not a statement) returning a promise for the module's *namespace object*. It triggers the same construct→link→evaluate pipeline for any not-yet-loaded subgraph, but lazily and asynchronously. It accepts computed specifiers, runs anywhere, and is the basis for code-splitting. The returned namespace object's properties are the same live bindings as static imports.

## See Also
- [[module-systems]]
- [[module-resolution]]
- [[async-await]]
- [[promises-internals]]
- [[01-programming-foundations/languages/javascript/execution-model/runtime|JS Runtime]]
- [[10-frameworks-and-stacks/node-js/modules/esm-in-node|ESM in Node]] — Node's ESM specifics
