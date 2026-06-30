# React Review Questions

A senior-level self-test on React internals, fundamentals, and design philosophy (React 19). Questions only — answer from your own mental model first, then verify against the linked notes.

## Philosophy

- Why does `UI = f(state)` *require* render to be pure and idempotent, and what concrete capabilities break if a component mutates during render?
- How does declarative rendering make reconciliation, time-travel, and concurrent rendering possible — and which of these is impossible against imperative DOM code, and why?
- Why did React choose composition over inheritance for UI reuse? What specific failure modes of mixins motivated their removal?
- Distinguish *composition* from *configuration* in component API design. When does adding another boolean prop become an anti-pattern?
- Hooks replaced HOCs and render props for which axis of reuse, and which axis of composition did they *not* replace?
- Why was two-way data binding rejected? What class of bugs does unidirectional flow structurally prevent?
- "Single source of truth" — why is copying a prop into state and letting it drift a violation, and what are the two correct alternatives?
- In what sense are `useEffect` and refs admissions that not everything is declarative? How do you decide what belongs in an effect?

## Elements / VDOM

- Precisely distinguish an element, a component, and a fiber. Which one holds `useState` data, and why can't it be the element?
- What does `<Counter/>` evaluate to, and when (and by whom) is `Counter` actually called? Why does calling `Counter(props)` directly cause problems?
- Why is defining a component inside another component a remount bug? Trace it through `type` identity.
- How does passing JSX through `props.children` enable a reconciliation bailout? What property of elements makes this work?
- What is and isn't stored in the VDOM/fiber tree? Where does live DOM state (focus, scroll, selection) live, and what's the implication?
- Refute the claim "the virtual DOM is fast." State the actual cost model in terms of render, reconcile, and commit.
- How do Svelte and Solid avoid a runtime diff, and what does React give up — and gain — by keeping a coarse-grained VDOM/fiber model?
- What changed about `ref` and the element brand symbol in React 19, and why is the "element is an immutable description" model unaffected?

## Fiber & Scheduling

- What problem did the Fiber rewrite solve that the old stack reconciler could not? Why was the call stack the obstacle?
- Explain the two phases of work: the interruptible render/reconcile phase and the synchronous commit phase. What must be true of each?
- What are the `child`, `sibling`, `return`, and `alternate` pointers on a fiber, and how does double-buffering (current vs work-in-progress) work?
- How does priority/lane-based scheduling decide what to render first, and how can a high-priority update interrupt an in-progress low-priority render?
- Why is it safe to throw away a partially completed render? Which guarantees about your component code make that safe?
- What is "tearing," why does concurrent rendering make it possible, and how does `useSyncExternalStore` prevent it?

## Hooks

- Why must hooks be called in the same order every render? Tie this concretely to how hook state is stored on the fiber.
- What does the dependency array actually control for `useEffect`, `useMemo`, and `useCallback`, and what equality check compares deps?
- Distinguish `useEffect`, `useLayoutEffect`, and `useInsertionEffect` by *when* in the commit cycle each fires and what each is for.
- Why does StrictMode double-invoke renders and re-run effects (mount→unmount→mount) in development? What bugs is it designed to surface?
- Explain `useState`'s functional updater form. When is `setState(s => ...)` required for correctness rather than convenience?
- What is a stale closure in a hook, and which mechanisms (deps, refs, functional updates) resolve each variant?
- What does `useRef` give you that `useState` doesn't, and why does mutating a ref not trigger a render?
- What problem does `useReducer` solve over `useState` beyond style preference, and how does it interact with batching?

## Concurrent Features

- What does `useTransition` mark, and how does a transition update differ in priority from an urgent update like a keystroke?
- How does `useDeferredValue` differ from debouncing? What does it defer and what does it not?
- Explain how Suspense suspends: what does a component "throw," and how does the nearest boundary coordinate fallback vs revealing content?
- What is automatic batching in React 18+, and how does it differ from the pre-18 behavior across async boundaries and event handlers?
- Why can concurrent rendering call your render function multiple times for a single committed update, and what does that demand of your code?
- What is `useSyncExternalStore` for, and why did external stores need a dedicated hook once concurrent rendering shipped?

## Server Components

- What is the fundamental boundary between a Server Component and a Client Component? What can each do that the other cannot?
- What is the RSC payload (the serialized output), how does it differ from HTML, and how does the client reconcile it into the existing tree?
- Why can Server Components reduce bundle size, and what specifically never ships to the client?
- How does the `"use client"` directive define the boundary, and what are the serialization constraints on props crossing from server to client?
- How do Server Components compose with Suspense and streaming to progressively reveal content?
- What are Server Actions (`"use server"`), and how do they change the data-mutation model compared to a client-fetch-then-POST flow?
- How do `use(promise)` and the React 19 form/action hooks (`useActionState`, `useFormStatus`, `useOptimistic`) fit the server/client data lifecycle?
