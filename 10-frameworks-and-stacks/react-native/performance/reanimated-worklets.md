# Reanimated Worklets

## Reanimated 2/3 Architecture

Reanimated 1 routed every animation update through the JS bridge, so animations janked whenever the JS thread was busy. Reanimated 2 introduced **worklets**: JS functions compiled and executed on the UI thread via JSI, bypassing the bridge entirely. Reanimated 3 keeps the same model but rewrites core parts for the New Architecture, adds layout animations, and improves Fabric interop. The worklet model is what makes smooth 60/120 FPS animations possible even under JS-thread contention.

## Worklets as JS Running on the UI Thread

A **worklet** is a function marked `'worklet';` — either explicitly, or implicitly because it is passed to a Reanimated API that wraps it. A Babel plugin extracts worklet bodies and their captures at bundle time; at runtime, a second Hermes (or JSC) instance lives on the UI thread and executes them there. It is a *different VM* — different globals, module cache, and memory space. You cannot share mutable state with the React VM except through shared values.

## Worklet Runtime vs React Runtime Boundary

Two Hermes instances, two module systems, two memory spaces. The UI runtime cannot `require()` arbitrary modules — only worklet-safe globals (shared values, Reanimated primitives, and a small standard library). Native module calls require `runOnJS` back to the React side. This is why idiomatic worklet code is small, focused, and free of third-party imports.

## Shared Values

**`useSharedValue(initial)`** returns a `{ value: T }` object backed by a native cell accessible from both runtimes. Writes are synchronized; reading `.value` in a worklet returns whatever was last written from either side, with UI-runtime writes typically winning. Keep operations on `.value` atomic — don't destructure it, and don't treat it as a regular variable. Complex types (objects, arrays) work, but assignment is a deep copy.

## useAnimatedStyle and useAnimatedProps

**`useAnimatedStyle`** compiles its callback to a worklet that re-runs on every frame in which a shared value it reads changes; the returned style applies to native views via a fast path that never crosses back to React:

```jsx
const style = useAnimatedStyle(() => ({ opacity: progress.value }));
```

**`useAnimatedProps`** is the same trick for non-style props — SVG `d`, `TextInput` value, scroll offsets — and is essential for continuously updating a prop that would otherwise force JS re-renders.

## runOnUI and runOnJS

**`runOnUI(fn)(...args)`** schedules a worklet on the UI runtime — use it for setup before the next frame. **`runOnJS(fn)(...args)`** posts back to the React runtime, typically to trigger `setState` or navigation. Both serialize their arguments (deep copy), so you cannot pass React refs, functions not marked as worklets, or closures with non-serializable captures. Forgetting `runOnJS` when calling a React callback inside a worklet causes silent hangs or crashes.

## Closure Capture Rules in Worklets

The Babel plugin analyzes the worklet body and captures referenced outer-scope variables, serializing them and re-injecting them into the UI runtime at invocation. Captures are *snapshotted*: if you capture a regular variable and the JS side mutates it later, the worklet still sees the snapshot. This is a frequent gotcha — memoize callbacks used as worklets, or move mutable state into shared values.

## Integration with Gesture Handler

`react-native-gesture-handler` v2 produces gesture events directly inside worklets, so an update like `Gesture.Pan().onUpdate(e => { translateX.value = e.translationX })` never crosses the bridge. This is the idiomatic way to build 60 FPS swipeable sheets, drawers, and carousels. The handler state machine and the worklet runtime share the same UI thread, so there is zero latency between a gesture update and the animation update.

## Frame Callbacks

**`useFrameCallback(({ timestamp, timeSincePreviousFrame }) => ...)`** runs a worklet every frame — useful for continuous animations driven by time or physics. Use the `timestamp` argument rather than `Date.now()` for consistent interpolation. Active frame callbacks keep the VSync loop hot, so pause them with `setActive(false)` when the animation isn't visible; leaving many active has a real battery cost.

## Layout Animations

**`Layout`**, `FadeIn`, `SlideInLeft`, `FadeOut`, and similar primitives auto-animate enter, update, and exit. They hook into the view lifecycle and interpolate on the UI runtime. They are good for simple entrance and exit; for choreographed sequences (stagger, multi-phase) compose them with `withSequence`/`withDelay`. Layout animations interact poorly with some list virtualizers — verify with FlashList specifically.

## Entering and Exiting Transitions

The **`entering`** and **`exiting`** props on `Animated.View` accept layout-animation specs. Exiting is tricky: React marks the view for removal, but Reanimated holds it long enough to run the animation. Conflicts with navigation (the screen is unmounting) are a common source of "animation cut short" bugs — use navigation-aware alternatives like `react-native-screens` native transitions for cross-screen effects.

## JSI Scheduling and Work Coordination

Reanimated uses JSI's `ShareableValue` and a custom scheduler to move data between runtimes without the bridge. The UI runtime runs at VSync cadence (60/120 Hz / ProMotion); heavy synchronous work in a worklet drops frames, just as blocking the React thread hurts interactions. If expensive math in a worklet causes jank, simplify it or offload with `runOnJS` to a background queue, then post the result back.

## Memory and Lifetime of Shared Values

A shared value lives as long as its owning `useSharedValue` hook's component stays mounted; on unmount, Reanimated releases the native cell. Worklets that capture shared values keep them alive through the capture graph, so orphaned worklets (attached to detached animators or leaked subscriptions) can leak the cell. `cancelAnimation(sharedValue)` terminates in-flight drivers explicitly — use it in cleanup to avoid ghost animations.

## Limitations: No Synchronous React State Access

Calling `setState` from a worklet requires `runOnJS(setX)(value)`, which fires asynchronously on the React runtime's next microtask. The same applies to navigation actions, `fetch`, promises, and most third-party libraries. Design animations to own their state in shared values and sync back to React only at discrete commit points (gesture end, animation complete).

## Debugging Worklets

`console.log` inside a worklet prints from the UI runtime and still reaches Metro, though stack traces reference transformed code. Reanimated 3 improved sourcemap quality for worklets, and breakpoints in Chrome/Hermes DevTools mostly work. The Reanimated Babel plugin emits diagnostic output when it fails to extract a worklet — common when dynamic-code patterns (reflection, indirect calls) confuse its static analysis.

## Performance Ceiling vs the Animated API

Legacy `Animated` with `useNativeDriver: true` also runs on the native side, but only for a fixed set of props (transforms, opacity). Reanimated worklets animate *anything* computable, including layout props that the legacy driver rejects. The ceiling is the UI thread itself — too many simultaneous worklet animations will jank it. If frame drops appear only with many concurrent animations, profile with Xcode Instruments (Time Profiler on the UI thread) or `systrace`.

## See Also
- [[gesture-handler]] — gestures drive worklet animations
- [[thread-delegation]] — worklets run off JS thread
- [[animation-optimization]] — worklets are the optimization tool
- [[re-renders]] — worklets avoid render cycles
