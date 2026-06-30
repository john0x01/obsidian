# Memory Management

## RN Memory Surfaces (JS Heap, Native Heap, GPU Memory)

An RN app occupies at least three distinct memory regions. The *JS heap* (Hermes or JSC) holds JS objects, closures, and module-level state; visible in the Hermes profiler. The *native heap* holds native-module state, bridged objects, ObjC/Java objects retained by native code, and image buffers. *GPU memory* holds decoded textures ready for the compositor (IOSurface on iOS, shared-memory bitmaps on Android). An iOS OOM almost always hits native+GPU before JS; Android can hit either. Profiling one surface in isolation misses the full picture.

## Hermes GC vs JavaScriptCore GC

Hermes uses a generational GC (young nursery + old gen) with a concurrent mark phase; JSC is similar but with different tuning and a more mature inline-caching story. Hermes's TTI win comes partly from smaller bytecode, partly from a faster initial GC cycle. Both are invisible in release, but the Hermes inspector shows pause durations and allocation sites — look there when GC-induced frame drops appear. Tune via `--gc-min-heap`/`--gc-max-heap` for unusual workloads.

## Image Cache Bloat

`Image` caches decoded bitmaps per process; the cache can balloon into hundreds of MB on a heavy feed. FastImage uses SDWebImage (iOS) / Glide (Android), both of which expose cache size controls. expo-image adds a disk-first policy that keeps the memory cache smaller. Always set size limits appropriate to your hardware target — unbounded caches are the #1 cause of mobile OOMs, and they're invisible until a slow-memory device starts evicting.

## Listener Leaks (NativeEventEmitter Subscriptions)

`addListener` returns a subscription; not calling `.remove()` in cleanup leaks the subscription and every variable its closure references. Idiomatic pattern: `useEffect(() => { const sub = emitter.addListener(...); return () => sub.remove(); }, [])`. Avoid `emitter.removeAllListeners(event)` — it also removes listeners added by libraries and causes "my feature silently stopped working" bugs.

## Timer Leaks (setInterval, setTimeout Not Cleared)

Every `setInterval` with a closure holds that closure until `clearInterval`. A navigation-away without cleanup keeps the interval firing and the state alive. `useEffect` cleanup is mandatory. For long-lived polling/heartbeat timers, guard against recreation when deps change — otherwise you accumulate ghost intervals that all fire together.

## Navigation Stack Retention

Stack navigators by default keep previous screens mounted so the back gesture shows the prior content. Deep stacks retain JS state, images, and subscriptions per screen. Strategies: `react-native-screens` with `enableScreens(true)` so the native container unmounts invisible screens; `detachInactiveScreens` on navigators; manual cache clear on `blur`. Without these, a "back-forward through 10 screens" session holds 10× the memory of a single screen.

## Long List Retention and Windowing

Virtualization unmounts off-screen items; the *data array* stays. Infinite feeds grow unboundedly unless you implement windowed data — drop items more than N screens from visible. Image components inside list items also keep decoded bitmaps alive in cache even after item unmount; pair list windowing with image-cache eviction to actually reclaim memory.

## Android Bitmap Memory Pressure

Android decodes images to `ARGB_8888` by default — a 4K photo is ~33 MB in memory regardless of JPEG size. Low-memory devices die well before the heap budget is hit. Mitigations: specify width/height so the loader downsamples at decode, use `resizeMode` aggressively, prefer `RGB_565` when alpha isn't needed, and use Glide's `downsampleStrategy` tuning. `BitmapCounter` in Android Profiler surfaces the bitmap pool size.

## Native Module Leaks

A native module holding JS callbacks (via `RCTPromiseResolveBlock`, `Callback`, retained event emitter blocks) pins those callbacks and their closures until release. Common bug: resolving a promise twice, or forgetting to resolve on an error path. On the native side, circular retention between Objective-C delegates and Swift closures is also common — Xcode's Memory Graph Debugger finds these cycles fast.

## Closure Retention in React Components

A `useCallback` with stale deps retains the initial render's entire closure for the component's lifetime, including DOM/native refs. More insidiously, any function stored in a ref or passed to a long-lived subscription closes over its capture set. For subscriptions, `useRef` with a callback identity often beats `useCallback` because it avoids stale-deps entirely — the ref always points at the latest function.

## Memory Profiling (Xcode Instruments, Android Profiler)

iOS: Xcode Instruments → Allocations shows retained objects by class with full backtraces; Leaks catches straightforward cycles; VM Tracker shows dirty memory by region. Android: Android Studio Profiler → Memory has heap dumps + allocation tracking; `adb shell dumpsys meminfo <pkg>` gives a quick PSS snapshot. Correlate platform-level memory with Hermes heap (via its inspector) to know which surface is bloating.

## didReceiveMemoryWarning and onLowMemory Handling

iOS sends `applicationDidReceiveMemoryWarning` and Android sends `onLowMemory`/`onTrimMemory` before killing your process. RN forwards these via `AppState` and `DeviceEventEmitter` (platform-specific event names). Use them to clear image caches, drop off-screen data, trim in-memory stores. Many apps ignore the warning and then wonder why they're terminated — on iOS especially, the warning usually arrives ~1–2 seconds before termination.

## Per-Screen Memory Budgets

Senior teams assign memory budgets per screen — feed X MB, chat Y MB — and enforce via profiling reviews. Without budgets, memory bugs are invisible until users complain. Track "peak RSS after 5 minutes of feed scrolling" as a regression metric in CI; a 30% increase without feature justification is a blocker, not a nice-to-have.

## Tracking RSS and PSS on Android

RSS (Resident Set Size) counts shared libraries multiple times across processes; PSS (Proportional Set Size) splits shared memory fairly. For app-specific decisions, PSS is the honest number. `adb shell dumpsys meminfo` reports both. Google Play's Android Vitals uses a proprietary measure that correlates best with PSS — if Vitals flags your app but local RSS looks fine, you're looking at the wrong number.

## Dev Mode vs Release Memory Differences

Dev builds include the RN Dev Menu, Flipper hooks, hot reload machinery, source maps, unminified JS — all pinned in memory. A dev build can use 2–3× the memory of release. Never benchmark memory on dev. Use release or profile-configured builds (Xcode "Release" scheme; Android `release` variant with debugging symbols if needed) for any number that matters.
