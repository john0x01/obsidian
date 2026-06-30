# JavaScript Review Questions

Senior-level self-test for the JavaScript track. Questions only — answer from your own mental model, then check against the linked notes. Grouped by domain.

## Execution Model & Internals

1. Walk through what happens when synchronous code runs: call stack, execution context, heap.
2. What is the event loop, and how do the macro-task and micro-task queues differ?
3. Where do `setTimeout` callbacks and resolved Promise callbacks each get queued, and which runs first?
4. How does the garbage collector decide what to reclaim?

## Language & Runtime

1. What does it mean that JS is single-threaded, and which parts of "the engine" are actually multithreaded?
2. How is a browser's event loop different from Node's (phases, `process.nextTick`, the check phase)?
3. What is a "realm," and how many exist in a page with three iframes and two Workers?

## Engine Internals (V8 / JIT)

1. Trace a function's journey through V8's pipeline: parser → Ignition bytecode → Sparkplug → Maglev → TurboFan. What triggers promotion between tiers?
2. What is a hidden class (a "Map"/"Shape") and why does adding properties in a consistent order matter for performance?
3. What is an inline cache (IC), and what is the difference between a monomorphic, polymorphic, and megamorphic call site?
4. What causes a deoptimization (deopt)? Give three concrete code patterns that trigger one and explain the bailout.
5. Why does mixing types in an array (e.g. smi + double + object) hurt? Explain V8's elements kinds and "packed vs holey" arrays.
6. What is the difference between a Smi and a HeapNumber, and when does V8 box a number?
7. How does on-stack replacement (OSR) let a hot loop get optimized without restarting the function?
8. Why is `arguments` (vs rest params) historically a deopt hazard, and what is "arguments leaking"?
9. What is a "shape transition tree" and how do `delete` and dynamically added properties degrade it?

## Language Semantics

1. Explain the abstract `ToPrimitive` algorithm and how `==` coercion differs from `===`. Predict `[] == ![]`.
2. Walk the prototype chain resolution for a property read. How do `[[Get]]`, `[[GetPrototypeOf]]`, and `Object.create(null)` interact?
3. Enumerate every rule that determines the value of `this`: default, implicit, explicit (`call`/`apply`/`bind`), `new`, arrow functions, and class fields.
4. What exactly does a closure capture — values or variable bindings? Explain the classic `var` loop bug and why `let` fixes it.
5. Distinguish `null` from `undefined` at the spec level. When does each appear (missing params, holes, `typeof`, JSON)?
6. What is the temporal dead zone, and how does it differ between `let`/`const` and `var` hoisting?
7. How do property descriptors (`writable`, `enumerable`, `configurable`) and getters/setters change object behavior? What does `Object.freeze` actually guarantee (and not)?
8. Explain boxing of primitives: why can you call `"abc".length` on a primitive string?
9. Why is `typeof null === "object"`, and why is `NaN !== NaN`? What is the difference between `===`, `Object.is`, and SameValueZero?
10. What are symbols for, and how do well-known symbols (`Symbol.iterator`, `Symbol.toPrimitive`, `Symbol.hasInstance`) hook into language operations?
11. How do `WeakMap`/`WeakRef`/`FinalizationRegistry` interact with GC, and why are their callbacks non-deterministic?

## Modules

1. Contrast ESM and CommonJS: load timing (static vs dynamic), `this` binding, top-level scope, and circular-dependency behavior.
2. What is a "live binding" in ESM, and how does it differ from CJS's value copy on `module.exports`? Show a case where the difference is observable.
3. Explain the three phases of ESM evaluation: construction (parse + resolve), instantiation (linking, allocating bindings), and evaluation. When are exports usable?
4. How does ESM handle a circular import that CJS would leave partially initialized? What does the importing module see?
5. What is top-level `await`, and how does it affect a module's place in the dependency graph and sibling evaluation order?
6. Why is tree-shaking possible with ESM but generally not with CJS? What breaks it (side effects, `sideEffects` flag, re-exports)?
7. What is the practical meaning of the `"type"` field, dual packages, and the `"exports"` map / conditional exports?

## Asynchrony

1. What is the precise ordering guarantee between microtasks and macrotasks, and when is the microtask queue drained?
2. Reproduce the internal state machine of a Promise: pending → fulfilled/rejected, the `[[PromiseFulfillReactions]]` lists, and why `.then` always schedules a microtask even on an already-settled promise.
3. Desugar an `async`/`await` function into promises and `.then` chains. Where exactly does execution suspend and resume, and on which queue?
4. Why does `await` on a non-promise (`await 42`) still cost a microtask tick? What changed in the spec to remove the extra ticks?
5. Explain the difference between `Promise.all`, `allSettled`, `race`, and `any` in terms of settlement and short-circuiting.
6. What is an unhandled rejection, when is it reported, and how can a late `.catch` change that?
7. How do generators relate to async/await? Sketch how a coroutine/`co`-style runner drives a generator with promises.
8. Why can a tight loop of `await`s starve rendering or I/O even though "async is non-blocking"?

## Concurrency & Parallelism

1. Why is Web Worker communication described as actor-like? What invariant does share-nothing message passing buy you?
2. What can and cannot cross a `postMessage` boundary? Explain the structured clone algorithm vs `JSON.stringify`.
3. What is a transferable object, what does "neutered/detached" mean, and how does transfer differ from clone in cost and semantics?
4. Distinguish dedicated, shared, and service workers by lifecycle, addressing, and intended use. Why is a Service Worker a poor place for in-memory state?
5. When you move CPU work to a Worker, are you buying throughput or responsiveness? How does Amdahl's law bound the win?
6. What does `SharedArrayBuffer` change about the JS concurrency model, and what hazards return?
7. State the JS memory model's guarantees: what ordering do `Atomics` operations provide, and what is the defined behavior of a non-atomic data race?
8. How do `Atomics.wait`/`notify` work, why does `wait` throw on the main thread, and what is `waitAsync` for?
9. Why was `SharedArrayBuffer` disabled after Spectre, and what exactly do COOP and COEP do to bring it back (`crossOriginIsolated`)?
10. How do WebAssembly threads use SAB to implement pthreads in the browser?

## Garbage Collection & Memory

1. Describe V8's generational GC: the young generation (Scavenge / semi-space copying) vs the old generation (mark-sweep-compact).
2. What is the generational hypothesis, and why does it justify a separate fast collector for new objects?
3. What makes a GC pause "stop-the-world," and how do incremental marking, concurrent marking, and lazy sweeping reduce it?
4. What is a write barrier and why is it needed for incremental/generational collection?
5. Define a memory leak in a GC'd language. Give four common leak sources (detached DOM, lingering closures, growing caches/maps, forgotten timers/listeners).
6. How do `WeakMap`/`WeakSet` help avoid leaks, and what are their limits (no enumeration, no size)?
7. How would you diagnose a leak with heap snapshots, retainers, and the three-snapshot technique?

## See Also
- [[runtime]] — the execution model under test
- [[garbage-collector]] — answers GC reclamation questions
- [[event-loops]] — task vs micro-task queues
- [[futures-and-promises]] — Promise callback ordering
- [[closures]] — closures study topic
- [[web-workers]] — concurrency questions
- [[shared-memory-and-atomics]] — memory-model questions
