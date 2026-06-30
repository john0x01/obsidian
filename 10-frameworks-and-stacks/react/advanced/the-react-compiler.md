# The React Compiler

The React Compiler (announced under the codename **"React Forget"**) is a build-time tool that **automatically memoizes** your components and hooks so React re-renders only what actually changed — eliminating most hand-written `useMemo`, `useCallback`, and `React.memo`. It is not a runtime; it's an optimizing compiler that rewrites your code during the build, inserting memoization a human would otherwise write by hand, more thoroughly and more correctly. The thesis: developers are bad at manual memoization (we forget it, over-apply it, or get dependency arrays wrong), so the compiler should own it.

## The problem it solves

By default React re-renders a component whenever its parent renders, regardless of whether its props changed. The cost is usually cheap, but for expensive subtrees the historical fix was manual memoization — wrapping values in `useMemo`, callbacks in `useCallback`, components in `memo`, and maintaining the dependency arrays by hand. This is tedious, error-prone, and noisy. Worse, manual memoization is *brittle*: one new prop that's an inline object or arrow function silently breaks a `memo` boundary downstream.

## How it works: inferring reactivity and inserting memoization

The compiler is a real compiler pipeline. It parses each component into an intermediate representation, builds the **data-flow graph** of every value, and infers which outputs depend on which inputs (props, state, hook results). It then emits code that **caches each computation and JSX subtree**, recomputing a given value only when its specific dependencies change.

Conceptually, the runtime result is a per-render **memoization cache** (a hidden slots array) keyed by position, against which each value's inputs are compared:

```jsx
// you write:
function Profile({ user, posts }) {
  const sorted = posts.sort(byDate);          // expensive
  return <Feed items={sorted} avatar={user.avatar} />;
}

// the compiler emits (sketch): cache slots, recompute `sorted` only
// when `posts` changes; preserve the previous <Feed/> element when
// neither `sorted` nor `user.avatar` changed → Feed skips re-render.
```

The key difference from manual memoization is **granularity and correctness**: the compiler can memoize sub-expressions and individual JSX nodes that you'd never bother wrapping by hand, and it derives the dependency set from the actual code rather than a hand-maintained array — so there are no stale-closure or missing-dependency bugs. It memoizes "everything worth memoizing" rather than the few spots you remembered.

## Its reliance on the Rules of React

The compiler can only safely cache and reorder work if your code obeys the **Rules of React** — the same rules that have always been required, now load-bearing:

- **Components and hooks must be pure** during render: same inputs → same output, no side effects, no mutation of props/state/external values while rendering.
- **Props, state, and hook return values are immutable** — you don't mutate them after creation. (Mutating `posts` in place, as in the sketch above, is actually a violation; the compiler assumes you don't.)
- **Hooks follow the Rules of Hooks** — called unconditionally, at the top level, in order.

When code follows these rules, memoizing a computation is semantically invisible — caching can't change behavior because pure functions of unchanged inputs produce unchanged outputs. When code *breaks* them (a render that mutates shared state, say), aggressive memoization can change observable behavior. The compiler therefore ships with a **static analyzer that bails out**: if it can't prove a component is safe, it **skips optimizing that component** and leaves it untouched rather than risk miscompiling. This fail-safe design is why adoption is low-risk — the worst case is "no optimization here," not "broken app." The companion **ESLint plugin** surfaces rule violations at authoring time so you can fix the code the compiler had to skip.

## What it changes about how you write React

- You stop writing `useMemo`/`useCallback`/`memo` for performance. They become rare, reserved for cases the compiler can't see (e.g. crossing a non-React boundary).
- You write **simpler, more direct** code — inline objects and arrow-function props stop being a performance concern, so the "hoist everything for referential stability" folklore largely dissolves.
- Following the Rules of React shifts from *good practice* to *prerequisite for optimization* — clean, pure components literally run faster.
- It does **not** change the mental model (`UI = f(state)`), the API, or how you reason about renders. It changes the *cost* of writing idiomatic code, removing the tax that pushed people toward manual micro-optimization.

## Current status

As of React 19, the compiler is available and stabilizing — it reached a release-candidate stage and is being rolled out for general adoption, integrated as a Babel plugin (and supported in the major build tools and the Next.js App Router). It's **opt-in** and **incremental**: you can enable it for the whole app or scope it to directories, and `'use no memo'` lets you exclude a specific component as an escape hatch during migration. The recommended on-ramp is to first get clean under the ESLint plugin, then enable the compiler.

## Senior pitfalls and misconceptions

- **"It makes everything faster automatically."** It removes unnecessary re-renders; it does not parallelize, doesn't speed up genuinely expensive single renders beyond memoizing them, and skips components it can't prove safe.
- **"I can stop caring about purity."** The opposite — the compiler *depends* on purity, and impure components are silently left unoptimized.
- **Ripping out all manual memoization preemptively.** Until the compiler is enabled and verified across the codebase, existing `useMemo`/`memo` still do their job; remove them as you adopt, not before.
- **Expecting it to fix architectural re-render problems.** Lifting state too high, or Context broadcasting to the whole tree, are structural issues the compiler can't paper over.

## See Also

- [[portals-and-refs]]
- [[error-boundaries]]
- [[state-architecture]]
- [[declarative-ui]]
