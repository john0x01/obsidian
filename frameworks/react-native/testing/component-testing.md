# Component Testing

## What Component Testing Covers in RN

Component tests render a React component in isolation (or a small subtree) and assert on its output and behavior. In RN they run under Jest using a JavaScript-level renderer — no simulator, no device, no native modules. The value is fast feedback on rendering logic, props → UI mapping, handlers, and state transitions. The limit is that nothing native is exercised — if your component's bug is in a TurboModule, component tests won't find it.

## react-native-testing-library Philosophy

`@testing-library/react-native` encourages assertions on *what the user sees*, not implementation details. Query by accessible role, label, or visible text first; `testID` only when nothing better works. This forces tests to be resilient to refactors — renaming a variable or reshaping the component tree shouldn't break tests. Enzyme-style "find by component class" tests are fragile and actively discouraged.

## Query Priorities (accessibilityLabel, text, testID)

Testing library recommends query order: `getByRole` → `getByLabelText` / `getByText` → `getByTestId`. The order aligns with accessibility — if users with screen readers can find the element, tests can too. Tests driven by accessibility queries double as an a11y audit. `testID` is a last resort; it couples tests to implementation details with no user benefit. 

## User-Event Simulation

`fireEvent.press(element)`, `fireEvent.changeText(input, 'text')` simulate user interactions. `user-event` (not yet fully parity for RN) produces richer interaction sequences. Most RN component tests use `fireEvent` directly. For complex gestures, component tests are the wrong layer — use Detox/Maestro.

## Snapshot Testing Trade-offs

`expect(tree).toMatchSnapshot()` serializes the rendered tree and compares to a committed file. Cheap to add, expensive to maintain: every intentional UI change requires reviewing and updating snapshots, and reviewers rubber-stamp large diffs. Reserve snapshots for genuinely visual output; prefer targeted assertions (`expect(screen.getByText('Submit')).toBeOnTheScreen()`) for logic.

## Mocking Native Modules

`jest.mock('react-native', () => ({ NativeModules: { MyModule: { ... } } }))` or auto-mocks for common packages (`@react-native-async-storage/async-storage`, `react-native-reanimated`). Many popular libraries ship their own mocks (check `jest.setup.js`). Without mocks, components that call native modules throw in the test environment — the Jest transform doesn't include native code.

## Mocking Navigation

React Navigation provides `@testing-library/react-native` helpers: wrap your component in `NavigationContainer` with a mock `navigationRef`. For screens that call `navigation.navigate`, pass a mock navigation prop. Don't try to test full navigation flows in component tests — that's E2E territory.

## Testing Hooks in Isolation

`renderHook(() => useMyHook())` from `@testing-library/react-hooks` (or `@testing-library/react` in newer versions) tests custom hooks without a containing component. Returns `result.current` for the hook's return value and re-renders on state changes. Essential for reusable hooks where the behavior is the unit of value.

## Testing with Context Providers

Components that consume context need a provider in tests. A reusable `renderWithProviders(component)` helper that wraps the target in all app-level providers (theme, navigation, query client, store) keeps individual tests tidy. Without it, every test rewires the same boilerplate.

## Async Testing (waitFor, findBy)

`findByText` waits for an element to appear (retries until timeout); `waitFor(() => expect(...).toBeTrue())` retries an assertion block. Both respect Jest's `testTimeout` config (default 5s). Use them for anything involving network, timers, or effect dependencies — the synchronous equivalents fail intermittently in async code.

## Timer Mocking (Jest fake timers)

`jest.useFakeTimers()` replaces `setTimeout`/`setInterval`/`requestAnimationFrame` with fakes advanced via `jest.advanceTimersByTime(ms)`. Essential for testing debounced/throttled behavior without waiting. Caveat: libraries that use `Date.now()` internally (Reanimated animations, some state managers) may behave unexpectedly — lock the clock with `jest.setSystemTime(date)`.

## Accessibility Assertions

`expect(element).toHaveAccessibilityState({ disabled: true })`; `expect(element).toHaveAccessibilityRole('button')`. Since testing-library pushes you toward accessibility-first queries, accessibility is already half-tested. Explicit assertions pin down role/label/state as part of the contract.

## Testing Reanimated Components

Reanimated ships a mock (`react-native-reanimated/mock`) that stubs worklets into regular functions. Add to Jest setup: `jest.mock('react-native-reanimated', () => require('react-native-reanimated/mock'));`. Animation timing isn't really tested — you verify initial render and end state, not intermediate frames. For real animation tests, you need E2E.

## Testing Gesture Handler Interactions

`react-native-gesture-handler/jestSetup` provides mocks. Gesture sequences are better tested in E2E; component tests at best verify that a handler is attached and that callbacks fire when manually invoked. Don't try to simulate gesture state machines in Jest — the interplay with worklets isn't accurately mocked.

## React Server Components Considerations

Not yet mainstream in RN (experimental). If you're testing components on the server boundary, traditional component tests cover the client half; the server half is a separate concern (happens at build time). Monitor RFC progress before investing in RSC-specific testing infrastructure.

## Comparing to Enzyme (Legacy)

Enzyme was the dominant library pre-testing-library; it allowed shallow rendering and component-internals access. Deprecated for RN — doesn't support React 18+. Don't start new projects with Enzyme; migrating is straightforward in most cases (query by text/role instead of component name).

## Coverage Tools for RN

Jest's built-in coverage (`--coverage`) emits Istanbul reports. Coverage numbers from component tests alone are misleading — native module code and platform-specific branches aren't instrumented. Set a floor (e.g., 70% statement coverage on JS-side logic) and treat it as a signal, not a goal.

## Test Performance

Jest tests should run in <1 second each on average; a 2000-test suite should finish in under 2 minutes with parallel workers. Slow tests usually come from unnecessary full-screen renders, missing mocks (an accidental real network call), or overuse of `waitFor` with long timeouts. Profile slow tests with `--verbose` and `--detectOpenHandles`.
