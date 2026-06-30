# Turbopack And SWC

Next.js replaced its JavaScript-based toolchain with Rust: **SWC** is the compiler that supplanted Babel (transform/transpile), and **Turbopack** is the bundler/dev server that is supplanting webpack. Both are bets on the same thesis — JS tooling is bottlenecked by the JS runtime itself, and a native, parallel, incrementally-cached pipeline in Rust changes the cost model of the inner loop. For a senior engineer the interesting part is not "it's faster," it's *why the architecture makes it faster* and what the current trade-offs are.

## SWC: The Compiler Layer

SWC (Speedy Web Compiler) parses and transforms TypeScript/JSX into runnable JavaScript. It does what Babel did — strip types, transpile JSX, apply syntax lowering, run Next.js's compile-time transforms (the `next/font` inlining, `styled-components`/emotion transforms, React Server Component directive handling, minification) — but it's written in Rust and is multi-threaded by design.

Why Rust beats Babel here:

- **No single-threaded JS event loop ceiling.** Babel runs in Node; parsing N files is serial-ish and GC-pressured. SWC parallelizes across cores natively.
- **No GC churn.** A Rust AST has predictable, manual-ish memory layout — no V8 garbage collector thrashing on millions of AST nodes.
- **One pass, one toolchain.** SWC owns parse → transform → emit, avoiding the plugin-boundary overhead Babel pays.

SWC ships *inside* Next.js — you generally don't configure it directly. The practical consequence: dropping a custom `.babelrc`/`babel.config.js` **opts your project out of SWC** and back onto Babel (slower), because Next.js can't guarantee SWC reproduces arbitrary Babel plugins. A common senior mistake is keeping a stray Babel config from years ago and silently losing the SWC fast path.

## Turbopack: The Bundler Layer

Turbopack is the incremental bundler and dev server. Its defining idea is **function-level incremental computation** built on a Rust framework (Turbo's engine, "Turbo Tasks"). Instead of treating the build as "given the module graph, produce the bundle," it models compilation as a graph of *memoized function calls*. Every unit of work — parse this file, resolve this import, transform this module, produce this chunk — is a cached function keyed by its inputs.

```
file change
   │
   ▼  invalidate only the affected cache cells
[parse f] → [transform f] → [chunk] → [HMR update]
   │            │
   └─ unchanged neighbors stay memoized; not recomputed
```

When a file changes, Turbopack invalidates only the cache cells whose inputs changed and recomputes the minimal downstream set — not the module graph, not the whole route. This is why HMR stays fast as the app grows: rebuild cost tracks the size of the *change*, not the size of the *project*. webpack, by contrast, tends to scale rebuild cost with graph size because its caching is coarser. The granularity is the whole story.

Two further architectural choices matter:

- **Lazy / demand-driven bundling in dev.** Turbopack compiles what a requested route actually needs, on demand, rather than eagerly building everything up front. Big apps reach an interactive dev server faster because you only pay for the page you opened.
- **Native + parallel.** Same Rust advantages as SWC — true threads, no GC stalls, tight memory layout for the dependency graph.

## Dev vs Build, and Current Status

The two phases have different priorities, and Turbopack's maturity differs across them:

- **Dev (`next dev --turbopack`)**: this is where Turbopack is most mature and where its incremental/lazy model pays off most — fast startup, fast HMR. In Next.js 15 Turbopack dev is **stable**.
- **Build (`next build --turbopack`)**: production builds add demands dev doesn't — full optimization, minification, deterministic output, the long tail of webpack ecosystem features. Production Turbopack builds have been stabilizing through the Next.js 15 line; confirm the exact stability status for your specific minor version rather than assuming, because this is the part that has been moving.

Until Turbopack reaches full webpack parity for build, **webpack remains the fallback** for production builds and for projects depending on webpack-specific loaders/plugins. SWC, by contrast, is already the default everywhere unless you force Babel.

## Trade-offs and Pitfalls

- **Plugin ecosystem gap.** A decade of webpack loaders/plugins and Babel plugins won't all have Turbopack/SWC equivalents. If your build leans on an exotic loader, you may be pinned to webpack or need an SWC plugin (SWC's Wasm plugin interface exists but its ABI has been version-sensitive — pin carefully).
- **The Babel trap.** Any `babel.config.js` silently disables SWC. Audit for it.
- **"Faster" is about the inner loop.** The headline wins are dev startup and HMR — the feedback loop you live in all day. Don't over-index on cold production build numbers when comparing.
- **Caching is the moat.** The reason to care architecturally: function-level memoization means correctness depends on *every input being part of the cache key*. When a transform reads something it didn't declare (env var, file outside the graph), incremental builds can go stale. This is a general hazard of incremental systems, not a Next.js bug — but it's why "delete `.next` and rebuild" remains the universal reset.

The philosophy across both tools: move the toolchain off the JS runtime into a native, parallel, aggressively-memoized engine so that build cost scales with *change size*, not *codebase size* — bending the developer feedback loop back toward instant.

## See Also

- [[node-vs-edge-runtime]]
- [[asset-optimization]]
- [[07-performance-engineering/compilation-optimization|Compilation Optimization]]
- [[08-quality-and-operations/devops-and-infrastructure/build-systems|Build Systems]]
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]
