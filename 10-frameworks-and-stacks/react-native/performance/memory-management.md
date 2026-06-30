# Memory Management

## RN Memory Surfaces: JS Heap, Native Heap, GPU Memory

An RN app occupies at least three distinct memory regions:

- The **JS heap** (Hermes or JSC) holds JS objects, closures, and module-level state; it is visible in the Hermes profiler.
- The **native heap** holds native-module state, bridged objects, Objective-C/Java objects retained by native code, and image buffers.
- **GPU memory** holds decoded textures ready for the compositor (IOSurface on iOS, shared-memory bitmaps on Android).

An iOS OOM almost always hits native and GPU memory before JS; Android can hit either. Profiling one surface in isolation misses the full picture.

## Hermes GC vs JavaScriptCore GC

**Hermes** uses a generational GC (young nursery + old generation) with a concurrent mark phase; **JSC** is similar but with different tuning and a more mature inline-caching story. Hermes's time-to-interactive win comes partly from smaller bytecode and partly from a faster initial GC cycle. Both are invisible in release builds, but the Hermes inspector shows pause durations and allocation sites — look there when GC-induced frame drops appear. Tune via `--gc-min-heap`/`--gc-max-heap` for unusual workloads.

## Image Cache Bloat

The `Image` component caches decoded bitmaps per process; on a heavy feed the cache can balloon into hundreds of MB. **FastImage** uses SDWebImage (iOS) and Glide (Android), both of which expose cache-size controls; **expo-image** adds a disk-first policy that keeps the memory cache smaller. Always set size limits appropriate to your hardware target — unbounded caches are the number-one cause of mobile OOMs, and they are invisible until a low-memory device starts evicting.

## Android Bitmap Memory Pressure

Android decodes images to **`ARGB_8888`** by default, so a 4K photo costs ~33 MB in memory regardless of its JPEG size. Low-memory devices die well before the heap budget is hit. Mitigations:

- Specify width and height so the loader downsamples at decode time.
- Use `resizeMode` aggressively.
- Prefer `RGB_565` when alpha isn't needed.
- Tune Glide's `downsampleStrategy`.

`BitmapCounter` in the Android Profiler surfaces the bitmap pool size.

## Listener Leaks: NativeEventEmitter Subscriptions

`addListener` returns a subscription; not calling `.remove()` in cleanup leaks the subscription and every variable its closure references. The idiomatic pattern ties the subscription to effect cleanup:

```js
useEffect(() => {
  const sub = emitter.addListener(...);
  return () => sub.remove();
}, []);
```

Avoid `emitter.removeAllListeners(event)` — it also removes listeners added by libraries, causing "my feature silently stopped working" bugs.

## Timer Leaks: setInterval and setTimeout Not Cleared

Every `setInterval` with a closure holds that closure until `clearInterval`. Navigating away without cleanup keeps the interval firing and its state alive, so `useEffect` cleanup is mandatory. For long-lived polling or heartbeat timers, guard against recreation when dependencies change — otherwise you accumulate ghost intervals that all fire together.

## Native Module Leaks

A native module holding JS callbacks — via `RCTPromiseResolveBlock`, `Callback`, or retained event-emitter blocks — pins those callbacks and their closures until release. A common bug is resolving a promise twice, or forgetting to resolve it on an error path. On the native side, circular retention between Objective-C delegates and Swift closures is also common; Xcode's Memory Graph Debugger finds these cycles fast.

## Closure Retention in React Components

A `useCallback` with stale dependencies retains the initial render's entire closure for the component's lifetime, including DOM and native refs. More insidiously, any function stored in a ref or passed to a long-lived subscription closes over its capture set. For subscriptions, `useRef` with a callback identity often beats `useCallback` because it avoids stale dependencies entirely — the ref always points at the latest function.

## Navigation Stack Retention

Stack navigators by default keep previous screens mounted so the back gesture can show the prior content. Deep stacks therefore retain JS state, images, and subscriptions per screen. Strategies to reclaim this:

- Use `react-native-screens` with `enableScreens(true)` so the native container unmounts invisible screens.
- Set `detachInactiveScreens` on navigators.
- Clear caches manually on `blur`.

Without these, a "back-and-forward through 10 screens" session holds 10× the memory of a single screen.

## Long List Retention and Windowing

Virtualization unmounts off-screen items, but the *data array* stays. Infinite feeds grow unboundedly unless you implement windowed data — dropping items more than N screens from visible. Image components inside list items also keep decoded bitmaps alive in cache even after the item unmounts; pair list windowing with image-cache eviction to actually reclaim memory.

## didReceiveMemoryWarning and onLowMemory Handling

iOS sends `applicationDidReceiveMemoryWarning` and Android sends `onLowMemory`/`onTrimMemory` before killing your process. RN forwards these via `AppState` and `DeviceEventEmitter` (the event names are platform-specific). Use them to clear image caches, drop off-screen data, and trim in-memory stores. Many apps ignore the warning and then wonder why they are terminated — on iOS especially, the warning usually arrives ~1–2 seconds before termination.

## Memory Profiling: Xcode Instruments and Android Profiler

On **iOS**, Xcode Instruments → Allocations shows retained objects by class with full backtraces; Leaks catches straightforward cycles; VM Tracker shows dirty memory by region. On **Android**, Android Studio Profiler → Memory provides heap dumps and allocation tracking, and `adb shell dumpsys meminfo <pkg>` gives a quick PSS snapshot. Correlate platform-level memory with the Hermes heap (via its inspector) to know which surface is bloating.

## Tracking RSS and PSS on Android

**RSS** (Resident Set Size) counts shared libraries multiple times across processes; **PSS** (Proportional Set Size) splits shared memory fairly. For app-specific decisions, PSS is the honest number. `adb shell dumpsys meminfo` reports both. Google Play's Android Vitals uses a proprietary measure that correlates best with PSS — if Vitals flags your app but local RSS looks fine, you are looking at the wrong number.

## Per-Screen Memory Budgets

Senior teams assign **memory budgets** per screen — feed X MB, chat Y MB — and enforce them via profiling reviews. Without budgets, memory bugs stay invisible until users complain. Track "peak RSS after 5 minutes of feed scrolling" as a regression metric in CI; a 30% increase without feature justification is a blocker, not a nice-to-have.

## Dev Mode vs Release Memory Differences

Dev builds pin the RN Dev Menu, Flipper hooks, hot-reload machinery, source maps, and unminified JS all in memory, so a dev build can use 2–3× the memory of release. Never benchmark memory on a dev build. Use release or profile-configured builds — the Xcode "Release" scheme, or the Android `release` variant with debugging symbols if needed — for any number that matters.

## See Also
- [[03-computer-systems/operating-systems/memory-management|Memory Management (OS)]] — underlying OS memory model
- [[list-and-virtualization]] — windowing caps memory
- [[image-pipeline]] — image caches dominate memory
- [[re-renders]] — leaked closures retain memory
