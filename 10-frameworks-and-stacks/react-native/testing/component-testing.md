# Component Testing

## What Component Testing Covers in RN

**Component tests** render a React component in isolation (or a small subtree) and assert on its output and behavior. In React Native they run under Jest using a JavaScript-level renderer — no simulator, no device, no native modules. Their value is fast feedback on rendering logic, props-to-UI mapping, handlers, and state transitions. Their limit is that nothing native is exercised: if a component's bug lives in a TurboModule, component tests won't find it.

## react-native-testing-library Philosophy

**`@testing-library/react-native`** encourages assertions on *what the user sees*, not on implementation details. Query by accessible role, label, or visible text first; reach for `testID` only when nothing better works. This forces tests to be resilient to refactors — renaming a variable or reshaping the component tree shouldn't break a test. Enzyme-style "find by component class" tests are fragile and actively discouraged.

## Query Priorities (accessibilityLabel, text, testID)

The library recommends a **query order**: `getByRole` → `getByLabelText` / `getByText` → `getByTestId`. This order aligns with accessibility — if a user with a screen reader can find the element, a test can too, so accessibility-driven queries double as an a11y audit. `testID` is a last resort: it couples tests to implementation details with no user benefit.

## User-Event Simulation

**`fireEvent`** simulates user interactions, e.g. `fireEvent.press(element)` and `fireEvent.changeText(input, 'text')`. The newer `user-event` API produces richer interaction sequences but does not yet have full parity for RN, so most RN component tests use `fireEvent` directly. For complex gestures, component tests are the wrong layer — use Detox or Maestro.

## Async Testing (waitFor, findBy)

Asynchronous output requires retrying queries. **`findByText`** waits for an element to appear, retrying until timeout; **`waitFor`** retries an assertion block, e.g. `waitFor(() => expect(...).toBeTrue())`. Both respect Jest's `testTimeout` config (default 5s). Use them for anything involving network, timers, or effect dependencies — the synchronous equivalents fail intermittently in async code.

## Timer Mocking (Jest Fake Timers)

**`jest.useFakeTimers()`** replaces `setTimeout`, `setInterval`, and `requestAnimationFrame` with fakes that are advanced via `jest.advanceTimersByTime(ms)`. This is essential for testing debounced or throttled behavior without waiting. Caveat: libraries that read `Date.now()` internally (Reanimated animations, some state managers) may behave unexpectedly — lock the clock with `jest.setSystemTime(date)`.

## Snapshot Testing Trade-offs

**Snapshot testing** serializes the rendered tree and compares it to a committed file via `expect(tree).toMatchSnapshot()`. It is cheap to add but expensive to maintain: every intentional UI change requires reviewing and updating snapshots, and reviewers tend to rubber-stamp large diffs. Reserve snapshots for genuinely visual output; prefer targeted assertions such as `expect(screen.getByText('Submit')).toBeOnTheScreen()` for logic.

## Accessibility Assertions

Because the library pushes you toward accessibility-first queries, accessibility is already half-tested. **Explicit accessibility assertions** pin down role, label, and state as part of the contract:

```js
expect(element).toHaveAccessibilityState({ disabled: true });
expect(element).toHaveAccessibilityRole('button');
```

## Mocking Native Modules

Components that call native modules throw in the test environment, because the Jest transform doesn't include native code. **Mocking native modules** stubs them out, either manually or via auto-mocks for common packages (`@react-native-async-storage/async-storage`, `react-native-reanimated`):

```js
jest.mock('react-native', () => ({ NativeModules: { MyModule: { /* ... */ } } }));
```

Many popular libraries ship their own mocks — check `jest.setup.js`.

## Mocking Navigation

Components that consume React Navigation need it mocked. Wrap the component in `NavigationContainer` with a mock `navigationRef`, or, for screens that call `navigation.navigate`, pass a mock navigation prop. Don't test full navigation flows in component tests — that's E2E territory.

## Testing with Context Providers

Components that consume context need a provider in tests. A reusable **`renderWithProviders(component)`** helper that wraps the target in all app-level providers (theme, navigation, query client, store) keeps individual tests tidy. Without it, every test rewires the same boilerplate.

## Testing Hooks in Isolation

**`renderHook`** tests a custom hook without a containing component — `renderHook(() => useMyHook())` from `@testing-library/react-hooks` (or `@testing-library/react` in newer versions). It returns `result.current` for the hook's return value and re-renders on state changes. This is essential for reusable hooks where the behavior is the unit of value.

## Testing Reanimated Components

**Reanimated** ships a mock that stubs worklets into regular functions. Add it to Jest setup:

```js
jest.mock('react-native-reanimated', () => require('react-native-reanimated/mock'));
```

Animation timing isn't really tested — you verify the initial render and the end state, not intermediate frames. For real animation tests, you need E2E.

## Testing Gesture Handler Interactions

**`react-native-gesture-handler/jestSetup`** provides the necessary mocks. Gesture sequences are better tested in E2E; in component tests you can at best verify that a handler is attached and that callbacks fire when manually invoked. Don't try to simulate gesture state machines in Jest — the interplay with worklets isn't accurately mocked.

## Coverage Tools for RN

Jest's built-in coverage (`--coverage`) emits **Istanbul reports**. Coverage numbers from component tests alone are misleading, because native module code and platform-specific branches aren't instrumented. Set a floor (e.g. 70% statement coverage on JS-side logic) and treat it as a signal, not a goal.

## Test Performance

Jest tests should average under one second each; a 2000-test suite should finish in under two minutes with parallel workers. Slow tests usually stem from unnecessary full-screen renders, missing mocks (an accidental real network call), or overuse of `waitFor` with long timeouts. Profile slow tests with `--verbose` and `--detectOpenHandles`.

## Comparing to Enzyme (Legacy)

**Enzyme** was the dominant library before testing-library; it allowed shallow rendering and access to component internals. It is deprecated for RN and doesn't support React 18+. Don't start new projects with it; migration is straightforward in most cases — query by text or role instead of by component name.

## React Server Components Considerations

**React Server Components (RSC)** are not yet mainstream in RN (still experimental). When testing components on the server boundary, traditional component tests cover the client half; the server half is a separate, build-time concern. Monitor RFC progress before investing in RSC-specific testing infrastructure.
