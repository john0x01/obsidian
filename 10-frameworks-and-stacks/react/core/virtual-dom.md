# Virtual DOM

The "virtual DOM" (VDOM) is React's **in-memory tree of element descriptions** that mirrors the structure of the UI you want, decoupled from the actual host DOM. It is *not* a faster DOM, not a clever DOM, and not a fixed React API surface — it's an implementation strategy: keep a cheap, plain-object representation of intended UI so React can diff it and apply the minimal real mutations. Much of the conventional wisdom about it ("VDOM is fast") is wrong; the value is *architectural*, not raw speed.

## What it IS and is NOT

- **IS:** a tree of immutable element objects (see [[elements-vs-components]]) produced by rendering, used as the input to reconciliation. In modern React the persistent, mutable mirror is actually the **fiber tree**, not a separate "virtual DOM" object — "VDOM" is best read as the conceptual category, while *fibers* are the concrete data structure.
- **IS NOT:** a literal shadow copy of the entire real DOM that React keeps in lockstep. It doesn't store layout, computed styles, or DOM-only state. It's a description of *structure and props*, not a clone of the rendered result.
- **IS NOT** inherently faster than direct DOM manipulation. A hand-optimized imperative patch will beat React on any single update. React trades a *constant overhead* (build description + diff) for never having to think about which mutations are minimal.

## Why a VDOM at all — three real reasons

1. **Decouple description from device.** Rendering produces *data*, so the same React tree can target the browser DOM, native views (React Native, via a different reconciler/renderer), canvas, PDFs, or test renderers. The element tree is renderer-agnostic; only the commit step is host-specific. This is the clean architectural payoff — the renderer is pluggable because UI is just a value.
2. **Batch and schedule.** Because mutations are *derived* from a diff rather than issued inline, React can coalesce many state updates into one diff+commit, avoiding intermediate layouts/reflows. It can also *defer* or *interrupt* the diff (concurrent rendering) precisely because building the description is pure and side-effect-free.
3. **Enable reconciliation.** You declare the whole intended tree each render; React derives the patch. The VDOM is the "previous intended tree" you diff the new one against — something that simply doesn't exist if you write imperative DOM code.

## The cost model (where seniors get it wrong)

The mental cost model is: **render is cheap, commit is the thing to minimize.**

```
render()      → build element tree   (cheap JS object allocation + your code)
reconcile()   → diff vs current fibers (O(n) over changed regions)
commit()      → apply minimal DOM ops (the expensive part: layout/paint)
```

"Re-rendering" a component means React *called your function and diffed the result* — it does **not** mean the DOM changed. A component can re-render every second and cause zero DOM mutations if its output diffs clean. The performance lever is therefore: (a) avoid *unnecessary* renders that allocate and diff for nothing, and (b) keep DOM-affecting changes minimal. The VDOM's overhead is the render+diff in (a); its benefit is correctness-by-default in (b).

The expensive failure mode is when a large subtree re-renders and re-diffs on a hot path. That's why `memo`, stable element identity, and (React 19) the **React Compiler's** automatic memoization exist — to skip render+diff for subtrees whose inputs are unchanged. None of this touches the DOM; it touches how much *describing and diffing* you do.

## Perspective: compile-time alternatives (Svelte, Solid)

The strongest critique of the VDOM is that the diff is *runtime work to recover information the compiler could have known statically*. Svelte and SolidJS take that position:

- **Svelte** compiles components into imperative DOM-update code at build time — no VDOM, no runtime diff. Each reactive variable knows exactly which DOM nodes it touches.
- **Solid** uses fine-grained reactivity (signals): an update mutates only the precise DOM nodes that depend on the changed signal; components run *once*, not on every update.

Their wins: no per-update diff overhead and smaller runtime. The VDOM's countervailing strengths: a *uniform, renderer-agnostic* model (one mental model across DOM/native/test), trivial "throw away and re-render the whole subtree" semantics that make interruptible **concurrent** rendering tractable, and a coarse-grained model that's simpler to reason about than a graph of fine-grained subscriptions. React 19 narrows the gap from the other side — the React Compiler removes much of the *manual* memoization tax while keeping the VDOM/fiber architecture, betting that compiler-assisted coarse-grained diffing plus concurrency is the better long-term tradeoff than fine-grained reactivity.

## Senior pitfalls and misconceptions

- **"VDOM is fast."** Reframe: VDOM makes *correct minimal updates the default* at a constant cost. Speed comes from avoiding work, not from the diff being magically quick.
- **"A render hit the DOM."** Usually false. Render → diff → maybe commit. Profile renders and commits separately.
- **"The VDOM stores the real DOM state."** It stores intended structure/props; live DOM state (scroll, focus, input selection) is *not* in it — which is exactly why you preserve such state via keys/refs, not by trusting the tree.
- **Conflating VDOM with fibers.** The diffable description is the element tree; the persistent runtime structure React actually mutates is the fiber tree.

## See Also

- [[reconciliation]]
- [[elements-vs-components]]
- [[declarative-ui]]
- [[jsx]]
- [[10-frameworks-and-stacks/react-native/architecture/rendering|RN Rendering]]
- [[10-frameworks-and-stacks/react-native/architecture/new-architecture-internals|RN New Architecture]]
