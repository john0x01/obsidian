# Re-Renders and Reconciliation

## Fabric Reconciliation Cost

Under the New Architecture, React reconciliation produces a shadow tree which is diffed C++-side by Fabric before the mount phase hits the main thread. JS-side reconciliation work is largely unchanged, but native mount (create/update views) is both faster and more observable — concurrent rendering can leave partially-committed trees. Practically: shallow re-renders that don't change tree shape are cheap under Fabric, while re-renders that toggle children in and out still force native view create/destroy. "Re-render" and "remount" have different costs; don't conflate them in profiling.

## React.memo, useMemo, useCallback Trade-offs

These aren't free. `React.memo` runs a shallow prop comparison on every parent render; for components with many props the comparison itself is measurable. Apply it to leaves that are expensive or frequently receive identical props, skip it for trivial leaves. `useMemo` and `useCallback` *preserve reference equality* — they don't skip work, they run every render. Apply only when a downstream `React.memo` or dependency array actually consumes that identity. Unprincipled application of all three everywhere is common and makes things slower.

## Component Recycling in FlashList / FlatList

FlashList recycles mounted components, so "render" ≠ "mount" — a single component instance will render many times with different item data. This breaks the model that unmount is the cleanup point. Treat recycled components as long-lived and clean up via `useEffect` keyed on the item ID, not via unmount. Subscriptions, animations, and timers all need explicit cancellation on item change.

## Render Scheduling and Priorities

React 18 concurrent rendering lets low-priority updates yield. RN picks this up under the New Architecture with Hermes. `useTransition`/`startTransition` mark updates as interruptible — important when a gesture is live and a list update shouldn't block the response. `useDeferredValue` is the cheap version: defer a derived value so the expensive downstream render runs when the thread has time. Urgent updates (input, navigation) still flush synchronously.

## Context Propagation Cost in RN

Every `useContext` consumer re-renders when the context value reference changes, regardless of whether the consumed slice changed. Theme/auth/navigation contexts that wrap the entire app amplify this: a single provider re-render cascades through hundreds of screens. Split contexts by change frequency (theme rarely, user state often) or use `use-context-selector` / Zustand / Jotai for slice-based subscription. A "one giant AppContext" almost always becomes a perf bottleneck.

## Navigation State Re-Renders

React Navigation stores state in a root context; every state change re-renders consumers unless you read with selectors like `useNavigationState(s => s.routes.length)` that return scalar values. Reading the full navigation object in a deep tree keeps everything subscribed. Prefer `useFocusEffect` for side effects, `navigation.addListener('focus', ...)` for imperative logic, and avoid reading navigation state in hot list-item paths.

## Redux / Zustand / Jotai Selector Patterns

Redux's `useSelector` re-runs on every dispatch; selectors that return new objects (`state => ({ x: state.x })`) always re-render. Use shallow equality or reselect memoization. Zustand's `useStore(s => s.x)` re-renders only when `x` changes by reference equality. Jotai's atom model avoids the problem: you subscribe only to the atoms you read. Senior advice: pick one store pattern and enforce it via lint rules rather than mixing.

## Avoiding Inline Objects and Functions as Props

`style={{marginTop: 10}}` allocates a new object on every render, forcing memoized children to re-render. Move to `StyleSheet.create` or module-level constants. Inline arrow handlers have the same problem — hoist with `useCallback` or, better, pass the raw ID and let the child call the stable handler itself. `data={[1, 2, 3]}` as a literal is also a new reference each render; these are surprisingly common perf regressions in refactors.

## React DevTools Profiler for RN

Connect with `npx react-devtools` and record a scroll or interaction. The flamegraph shows component render duration; "Ranked" mode sorts by cost. "Record why each component rendered" is the key feature — it attributes each render to a specific prop/state/context change. Production builds have the profiler disabled unless compiled in; for realistic numbers, build with `enableProfiler` on a release-variant config.

## why-did-you-render in RN

Patches `React.memo` and hooks to log reasons for unnecessary re-renders. High signal for identity-equality bugs (new objects, functions, arrays on every render) but noisy on first run. Dev-only — the patching overhead is substantial and would skew release profiles.

## List Item Render Cost Budgeting

Target ≤1ms per item render on scroll for 60 FPS (16.67ms per frame, multiple items render per frame). Profile a single item in isolation and multiply by batch size. If an item crosses 4–5ms, it can't keep up with fast scrolls regardless of virtualization — split the tree (lazy-render below-the-fold item children), reduce JSX depth, or defer decorative animations until the item settles.

## Screen-Level Mount / Unmount Cost

Switching tabs or pushing a screen mounts a large tree at once; this rarely shows in micro-profiling but appears as visible lag on low-end devices. Strategies: `InteractionManager.runAfterInteractions(() => ...)` to defer heavy work past the navigation animation, lazy-mount non-critical sections (analytics panels, below-the-fold modules), lean on `React.lazy`/`Suspense` to split the bundle by route.

## StyleSheet.create and Style Memoization

`StyleSheet.create` historically mapped styles to numeric IDs for the old bridge. Under Fabric, IDs still reduce the diff payload and — more importantly — provide stable references. Always prefer it over inline styles for list items. Style array composition (`[styles.base, isActive && styles.active]`) creates new arrays each render; memoize those too when feeding memoized children.

## Animated Values vs Re-Renders

`Animated.Value` (legacy) and `SharedValue` (Reanimated) don't trigger React re-renders when they change — updates flow to the native/UI thread directly. That's why Reanimated animations don't jank when the JS thread blocks. Resist mirroring animated values back into React state (`setValue(animated.__getValue())`) — it defeats the architecture. Sync back only at discrete moments: gesture end, animation completion.

## Concurrent Rendering Implications

React 18 concurrent features work under the New Architecture. `Suspense` boundaries show fallback UI while lazy-loaded screens fetch JS bundles or data, replacing ad-hoc loading state. `useTransition` marks updates non-urgent — a typeahead search that updates results via `startTransition` keeps the input responsive. Caveat: Reanimated worklets run outside React entirely, so concurrent boundaries don't affect them; gesture-driven animations are always "urgent" by construction.
