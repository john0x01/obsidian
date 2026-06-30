# Error Boundaries

An error boundary is a component that **catches errors thrown during rendering of its subtree** and renders a fallback instead of letting the whole tree unmount. It is React's answer to the fact that, by default, an uncaught render error crashes the *entire* root: since React 16, a throw during render tears down the whole component tree to avoid leaving the UI in a corrupt, half-rendered state. Boundaries let you contain that blast radius. They remain a **class-only** API — there is no hook equivalent — because the catching behavior depends on lifecycle methods.

## The two lifecycle methods

A class becomes an error boundary by implementing either or both:

```jsx
class Boundary extends React.Component {
  state = { error: null };
  static getDerivedStateFromError(error) {
    return { error };            // render phase: compute fallback state
  }
  componentDidCatch(error, info) {
    logToService(error, info.componentStack); // commit phase: side effects
  }
  render() {
    return this.state.error ? <Fallback error={this.state.error}/> : this.props.children;
  }
}
```

- **`getDerivedStateFromError`** runs during the *render* phase. It must be pure and return the next state — this is where you flip into fallback mode. Because it's render-phase, no side effects here.
- **`componentDidCatch`** runs during the *commit* phase. This is where logging belongs: it receives the error plus an `info` object whose `componentStack` tells you *which component* threw — invaluable, since the JS stack alone often points only into React internals.

## What boundaries catch — and what they don't

This is the defining subtlety. Boundaries catch errors thrown **synchronously during rendering, in lifecycle methods, and in constructors of the components below them.** They do **not** catch:

- **Event handlers** — an `onClick` throw happens outside React's render flow, on the browser's call stack. Use a plain `try/catch`.
- **Asynchronous code** — `setTimeout`, promises, `fetch().then(...)`. The throw escapes after render has long committed, so there's no boundary on the stack. (A common pattern: catch the rejection, store it in state, then `throw` it during the *next* render so a boundary can see it.)
- **Server-side rendering errors** — handled by the SSR API's own error channels, not the client boundary directly.
- **Errors in the boundary itself** — a boundary can't catch its own render error; it propagates to the next boundary up.

The mental model: a boundary is a `try/catch` around the **render phase**, nothing else. If the error doesn't arrive *on React's rendering call stack*, the boundary never sees it. This is the single most common misconception — engineers wrap a component expecting it to catch a failed `fetch` in an event handler, and it silently doesn't.

## Resetting

A boundary that has flipped to its fallback stays there until you reset its state. Two reset strategies:

- **Imperative** — pass a callback into the fallback that calls `this.setState({ error: null })`, then re-renders children. Risky if the underlying cause persists: it'll throw again immediately.
- **Key-based** — remount the boundary (or the children) by changing a `key` tied to the thing that should retry, e.g. `<Boundary key={routeId}>`. Changing the key discards the errored instance and mounts a fresh one. This is also how you auto-reset on navigation. Libraries like `react-error-boundary` formalize this with a `resetKeys` prop and an `onReset` callback.

## React 19 error-handling options

React 19 refines how *uncaught* and *caught* errors are surfaced through `createRoot`/`hydrateRoot` options:

- **`onCaughtError`** — fires when an error boundary *did* catch an error. Lets you centrally log handled errors.
- **`onUncaughtError`** — fires for errors no boundary caught (which then crash the root).
- **`onRecoverableError`** — fires when React recovered automatically, notably hydration mismatches it could patch over.

These give you a single observability seam instead of relying solely on per-boundary `componentDidCatch`. React 19 also reduced the noisy duplicate console logging of caught errors that earlier versions emitted.

## Suspense + error coordination

`<Suspense>` and error boundaries are complementary halves of declarative async UI: Suspense handles the **pending** state (a child suspended), a boundary handles the **rejected** state (a child threw). When data fetching throws a *real* error (not a suspension), it bubbles past the Suspense boundary to the nearest error boundary. The idiomatic pattern wraps the two together:

```jsx
<ErrorBoundary fallback={<Error/>}>
  <Suspense fallback={<Spinner/>}>
    <AsyncData/>      {/* suspends → Spinner; throws → Error */}
  </Suspense>
</ErrorBoundary>
```

With the `use` hook and async data, a rejected promise surfaces as a thrown error at the consuming component, so it lands in the enclosing error boundary — making "loading" and "failed" both declarative tree states rather than imperative flags you juggle.

## Senior pitfalls

- **Expecting event-handler/async errors to be caught.** They aren't; this is the number-one trap.
- **One root-level boundary only.** A single top boundary means any failure blanks the app. Place boundaries at meaningful seams (per route, per widget) so a failing panel doesn't take down the page.
- **Logging in `getDerivedStateFromError`.** It's render-phase and may run multiple times / be discarded — side effects belong in `componentDidCatch`.
- **Forgetting to reset.** Without a reset path, a transient error permanently bricks the subtree until full reload.

## See Also

- [[server-vs-client-components]]
- [[actions-and-forms]]
- [[portals-and-refs]]
- [[declarative-ui]]
