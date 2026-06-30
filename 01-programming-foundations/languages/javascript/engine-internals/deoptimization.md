# Deoptimization

Optimized JavaScript is *speculative*: the JIT compiles a function assuming the types and object shapes it observed so far will keep holding, and guards that assumption with cheap runtime checks. **Deoptimization (deopt, or "bailout")** is the controlled escape hatch — when a guard fails, V8 throws away the optimized machine code's path and resumes correct execution in the bytecode interpreter. It exists because correctness must never depend on a guess; deopt is what makes aggressive speculation *safe*. The senior skill is keeping your hot functions in the optimized state and recognizing when they're being kicked out of it.

## Why Optimized Code Gets Discarded

TurboFan/Maglev (see [[jit-compilation]]) emit code specialized to observed conditions, each protected by a guard. A guard failing — a *bailout* — triggers deopt. Common triggers:

- **Type change** — a parameter that was always a Smi (small integer) now arrives as a double, string, or object. `function f(x){ return x*2 }` optimized for integers deopts when first called with `"3"` or `1.5`.
- **Hidden-class / shape change** — an object reaches a call site with a different hidden class than the cached one, or its shape mutated after optimization (a field added, or `delete` dropping it to dictionary mode — see [[hidden-classes-and-inline-caches]]).
- **`undefined`/`null` where a value was assumed**, array-bounds or "holey" array access when packed was assumed, integer overflow turning a Smi into a heap number, `arguments` misuse, `try/catch` and `with` historically, etc.

There are two flavors. **Eager deopt** happens at a guard *as it executes*: V8 reconstructs the equivalent interpreter frame (locals, stack, the right bytecode offset) from the optimized frame — the inverse of on-stack replacement — and continues in Ignition without any observable difference except speed. **Lazy deopt** invalidates already-optimized code without unwinding immediately: some global assumption broke (e.g. a prototype was mutated, a function it inlined changed), so the code is marked invalid and dropped the next time control would enter it.

## The Deopt Loop: the Pathology to Fear

A single deopt is cheap and normal — the worry is the **deopt loop** (thrashing). V8 re-optimizes a hot function, the code hits the same disqualifying condition, deopts, re-optimizes, deopts again. Now you pay *both* the recompilation cost *and* run in the interpreter repeatedly — strictly worse than never optimizing. After enough churn V8 may mark a function **"disabled for optimization"** entirely, condemning it to the interpreter for good.

```
hot fn -> optimize (assume x is Smi)
       -> caller passes a double -> guard fails -> deopt
       -> re-optimize -> double again -> deopt ...   (thrash)
       -> V8 gives up: function permanently un-optimized
```

The classic cause: a function on a hot path that is *genuinely* polymorphic — fed different argument types or differently-shaped objects by different callers — so no single specialization is stable.

## Keeping Functions Optimizable

The defense is making your speculation-friendly assumptions actually true:

- **Type stability.** Call a function with consistent argument types. Don't reuse one function for ints in one branch and strings in another on a hot path; split it or normalize inputs at the boundary.
- **Shape stability.** Build objects with the same fields in the same order; finish initialization in the constructor; never `delete` (set to `null`); don't add properties to an object after it's been flowing through hot code. Keep arrays packed and of a single elements kind (all Smis / all doubles / all objects).
- **Monomorphic call sites.** Feed each call site one hidden class where you can; megamorphic sites can't be inlined and won't optimize well.
- **Avoid known de-optimizers on hot paths.** Materializing/leaking `arguments`, deeply mixed union types, mutating shared prototypes after warm-up. (Modern V8 optimizes far more than old "crankshaft killers" lists claim — verify against the current engine rather than trusting stale advice.)
- **Let cold code be cold.** None of this matters for run-once code; only profile-confirmed hot functions deserve the discipline.

## Tooling

Reason from evidence, not folklore — engine behavior changes release to release. V8 flags (via `node --flag`, or `d8`):

- **`--trace-opt`** — logs when functions are optimized (and to which tier), so you can confirm a hot function actually tiered up.
- **`--trace-deopt`** — logs every deopt with the function, bytecode offset, the *reason* ("wrong map", "not a Smi", "lost precision"…), and eager vs lazy. This is the primary instrument: a function appearing repeatedly here is your thrash.
- **`--trace-opt-verbose`** / **`--trace-ic`** — deeper detail on optimization decisions and inline-cache state transitions (mono→poly→mega), which usually *explains* the deopt reason.
- **`%OptimizeFunctionOnNextCall` / `%GetOptimizationStatus`** with `--allow-natives-syntax` — force-optimize a function and query its status in a microbenchmark to confirm it stays optimized under realistic inputs.
- **Chrome DevTools** surfaces deopt reasons in the Performance panel; flame charts that show a function running in the interpreter after warm-up are a red flag.

**The philosophy**: deopt is not a bug, it's the mechanism that lets a dynamically-typed language run at near-native speed without sacrificing correctness — optimize boldly, guard cheaply, fall back safely. Your job is to remove the *reasons* for repeated bailouts on the paths that matter, and to ignore the ones that don't.

## See Also
- [[jit-compilation]] — speculation and OSR that deopt reverses
- [[hidden-classes-and-inline-caches]] — shape/type feedback that, when violated, triggers deopt
- [[parsing-and-bytecode]] — the interpreter tier deopt falls back to
- [[v8-architecture]] — pipeline context for tier transitions
- [[07-performance-engineering/profiling|Profiling]] — finding hot functions to inspect
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]] — speculation as a JIT-only technique
