# iOS Profiling

## Xcode Instruments Overview

**Instruments** is Xcode's profiler suite (Product → Profile, `Cmd+I`). It launches the app attached to a chosen instrument — Time Profiler, Allocations, Leaks, Network, and others. It runs against debug or release builds, but for representative numbers, profile *release-config* builds: Debug has unoptimized JS, extra tracing, and different memory behavior. Nearly every serious iOS performance issue is diagnosed here, so it is worth learning properly.

## CPU and Thread Profiling

### Time Profiler

The **Time Profiler** samples the call stack periodically across all threads. The Call Tree view aggregates hot functions; filter by thread to isolate the JS thread (typically named `com.facebook.react.JavaScript`) or the main thread. "Invert Call Tree" sorts by self time to show which leaves actually consume CPU, and "Hide System Libraries" focuses on your code.

### Analyzing the JS Thread on iOS

In the Time Profiler, filter to the `com.facebook.react.JavaScript` thread. Deep stacks of JSC/Hermes internals are expected at the top, with your code below. `requestAnimationFrame`, `setTimeout` callbacks, and React rendering work all run on this thread. **JS-thread blocking** beyond a frame budget (~16 ms) is the universal source of dropped frames originating from JS.

### Analyzing the UI Thread (Main Queue)

The **main thread** handles native view creation, layout, and drawing. Under the Old Architecture, RN runs Yoga layout on a separate shadow thread and then commits on the main thread; under Fabric, commits are queued but still hit the main thread. Long main-thread tasks — file reads, synchronous native module calls on the main queue — directly cause jank. Profile this work with the Time Profiler plus the main-thread filter.

### System Trace

**System Trace** captures system-wide events: threads, locks, file I/O, virtual memory, and Grand Central Dispatch. It is best for understanding multi-thread contention — the JS thread blocked waiting for native I/O on the main thread shows up as a clear lock graph. It is heavier to record than the Time Profiler, so reserve it for specific investigations.

### os_signpost for Custom Events

Annotate your own code with **`os_signpost`** (or `OSLog` in newer SDKs) to produce named events in Instruments. RN's bridge and Fabric emit signposts for key operations such as commit and mount. Adding custom signposts around suspected hot paths ("feed load start", "feed load end") makes correlating JS-side events with platform-level traces straightforward.

### Hermes Profiler Integration

Enable the **Hermes sampling profiler** from JS, then save the trace:

```js
HermesInternal.enableSamplingProfiler();
// ... exercise the code path ...
HermesInternal.disableSamplingProfiler();
HermesInternal.dumpSampledTraceToFile(path);
```

Open the output in `chrome://tracing` or the Chrome DevTools Performance tab. It shows JS function samples correlated with timestamps, pairing with Instruments to explain "this JS function was hot while the UI thread was blocked".

## Memory Profiling

### Allocations

The **Allocations** instrument tracks every heap allocation with a full backtrace. Mark generations (reset counters) around a specific user action to see what was allocated during it — the classic "snapshot before, do thing, snapshot after" workflow that finds leaks. The Statistics tab grouped by class shows type-level counts, and anomalies (unexpectedly high counts) are usually the bug.

### Leaks

The **Leaks** instrument runs Allocations plus periodic leak detection for retain cycles and unreachable memory, reporting each leak with its allocation stack. It scores easy wins for ObjC/Swift cycles but does not catch JS-level leaks, which show up as a growing JS heap in the Hermes inspector rather than as ObjC leaks.

### Memory Graph Debugger

To use the **Memory Graph Debugger**, pause the debugger and click the Memory Graph button (the three-dots icon). It shows the object graph with retain counts, which is useful for finding retain cycles. Filter by class to see all live instances of a type: 1000 live instances of a screen's view controller means you are leaking screens.

## Rendering and I/O

### Core Animation FPS

The **Core Animation** instrument measures the render-server frame rate and lets you toggle diagnostic overlays:

- "Color Blended Layers" — excessive compositing.
- "Color Hits Green / Misses Red" — image caching effectiveness.
- "Color Offscreen-Rendered Yellow" — expensive composited effects.

For RN, offscreen rendering from shadows and rounded corners with `overflow: hidden` is a common culprit.

### Network Instruments

The **Network** instrument captures `URLSession`/`NSURLConnection` traffic with timing, headers, and payload. It is essential for comparing perceived slowness to actual request duration — often the "slow network" is actually slow rendering after a fast response. Export captures to share with backend teams.

### Disk Activity Profiling

The **File Activity** instrument captures reads, writes, and mmaps. Surprisingly often, app slowness is caused by synchronous disk access on a critical thread. Image caches, logs, and AsyncStorage (when misused) are common culprits.

## Energy and Launch

### Energy Impact and Battery Testing

The **Energy Log** instrument shows CPU, location, networking, and display energy per second. High-energy regions highlight unexpected background work such as polling or animations that do not pause when off-screen. Energy tests on device require unplugging — simulator energy data is not representative.

### Launch Time Measurement

Instruments has an **App Launch** template that captures the pre-main, main, and first-frame phases. Alternatively, set the `DYLD_PRINT_STATISTICS` environment variable in the scheme to log dyld timing. Launch performance breaks into dyld (library load), ObjC runtime init, and `main()` → `RCTRootView` init → first JS load → first commit; each phase can be profiled separately.

## Environment Caveats

### Simulator Limitations

The **simulator** runs x86_64 (Intel) or arm64 (Apple Silicon) — not the device's ARM. Its CPU is far faster, its GPU is desktop-class, and disk and network run at host speed. Performance numbers from the simulator are directionally correct for UI-thread / JS-thread patterns but useless for absolute performance. Always validate on device, preferably the lowest-spec one you support.

### Debugging Release-Config Builds

Debug builds have optimizations off, extra tracing on, and `__DEV__` checks live. For representative performance, profile **Release builds**. Create a "Profile" scheme that uses the Release configuration but keeps debug symbols — that is what Product → Profile uses by default. Do not optimize based on Debug numbers; they represent a different workload.

## See Also
- [[android-profiling]] — Android counterpart
- [[devtools]] — general debugging tooling
- [[startup-and-tti]] — profile cold-start
- [[10-frameworks-and-stacks/react-native/performance/memory-management|Memory Management (RN)]] — profile memory growth
