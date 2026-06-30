# React

Deep notes on how React works — philosophy, the reconciler (Fiber), the rendering model, hooks internals, state, and Server Components. Targets React 19. For mastery, not API lookup.

[← Back to Frameworks & Stacks](../README.md)

## Philosophy
- [Declarative UI](philosophy/declarative-ui.md) — `UI = f(state)` and what it buys you
- [Composition Model](philosophy/composition-model.md) — composition over inheritance
- [Unidirectional Data Flow](philosophy/unidirectional-data-flow.md)

## Core Model
- [Elements vs Components](core/elements-vs-components.md)
- [JSX](core/jsx.md) — the automatic runtime, `createElement`
- [Virtual DOM](core/virtual-dom.md) — what it is, and isn't
- [Reconciliation](core/reconciliation.md) — the diffing heuristics, keys

## Fiber (the Reconciler)
- [Fiber Architecture](fiber/fiber-architecture.md) — units of work, the fiber tree, `alternate`
- [Work Loop & Scheduling](fiber/work-loop-and-scheduling.md) — interruptible rendering, the Scheduler
- [Lanes & Priorities](fiber/lanes-and-priorities.md)

## Rendering
- [Render & Commit Phases](rendering/render-and-commit-phases.md) — double buffering, purity
- [Concurrent Rendering](rendering/concurrent-rendering.md) — tearing, `useSyncExternalStore`
- [Suspense](rendering/suspense.md)
- [Transitions](rendering/transitions.md)

## Hooks
- [Hooks Internals](hooks/hooks-internals.md) — the per-fiber linked list & dispatcher
- [Rules of Hooks](hooks/rules-of-hooks.md) — why they exist
- [State & Reducers](hooks/state-and-reducers.md) — the update queue, batching
- [Effects](hooks/effects.md) — synchronization, not lifecycle
- [Memoization](hooks/memoization.md) — `useMemo`/`useCallback`/`memo` and the compiler
- [Refs](hooks/refs.md)
- [Context](hooks/context.md)
- [Concurrent Hooks](hooks/concurrent-hooks.md) — `useTransition`, `useDeferredValue`, `use`

## State Management
- [State Architecture](state-management/state-architecture.md) — where state should live
- [External Stores](state-management/external-stores.md) — Redux/Zustand/Jotai & `useSyncExternalStore`

## Server Components
- [RSC Model](server-components/rsc-model.md)
- [Server vs Client Components](server-components/server-vs-client-components.md)
- [Actions & Forms](server-components/actions-and-forms.md)

## Advanced
- [Error Boundaries](advanced/error-boundaries.md)
- [Portals & Refs to the DOM](advanced/portals-and-refs.md)
- [The React Compiler](advanced/the-react-compiler.md) — automatic memoization

## Self-Test
- [Review Questions](review-questions.md)
