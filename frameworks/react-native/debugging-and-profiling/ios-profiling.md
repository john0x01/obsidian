# iOS Profiling

## Xcode Instruments Overview

Instruments is Xcode's profiler suite (Product → Profile, `Cmd+I`). It launches the app attached to a chosen instrument (Time Profiler, Allocations, Leaks, Network, etc.). Runs against debug or release; for representative numbers, profile *release-config* builds (Debug has unoptimized JS, extra tracing, and different memory behavior). Every serious iOS perf issue is diagnosed here — learn it properly.

## Time Profiler

Samples the call stack periodically across all threads. The Call Tree view aggregates hot functions; filter by thread to isolate the JS thread (typically named `com.facebook.react.JavaScript`) or main thread. "Invert Call Tree" sorts by self time — shows which leaves actually consume CPU. "Hide System Libraries" focuses on your code.

## Allocations

Tracks every heap allocation with full backtrace. Mark generations (reset counters) around a specific user action to see what was allocated during it — the classic "snapshot before, do thing, snapshot after" workflow finds leaks. The Statistics tab grouped by class shows type-level counts; anomalies (unexpectedly high counts) are usually the bug.

## Leaks

Runs Allocations plus periodic leak detection (retain cycles, unreachable memory). Reports found leaks with their allocation stack. Easy wins for ObjC/Swift cycles; doesn't catch JS-level leaks (those show up as growing JS heap in Hermes inspector, not ObjC leaks).

## Network Instruments

Captures URLSession / NSURLConnection traffic with timing, headers, payload. Essential when comparing perceived slowness to actual request duration — often the "slow network" is actually slow rendering after a fast response. Export captures for sharing with backend teams.

## Core Animation FPS

Measures render-server frame rate and lets you toggle overlays: "Color Blended Layers" (excessive compositing), "Color Hits Green / Misses Red" (image caching), "Color Offscreen-Rendered Yellow" (expensive composited effects). For RN, offscreen rendering from shadows and rounded corners with `overflow: hidden` is a common culprit.

## System Trace

System-wide event capture (threads, locks, file I/O, virtual memory, Grand Central Dispatch). Best for understanding multi-thread contention — the JS thread blocked waiting for a native I/O on the main thread shows as a clear lock graph. Heavier to record than Time Profiler; use for specific investigations.

## os_signpost for Custom Events

Annotate your own code with `os_signpost` (or `OSLog` in newer SDKs) to produce named events in Instruments. RN's bridge and Fabric emit signposts for key operations (commit, mount). Adding custom signposts around suspected hot paths ("feed load start", "feed load end") makes correlating JS-side events with platform-level traces straightforward.

## Hermes Profiler Integration

Enable Hermes sampling profiler via JS: `HermesInternal.enableSamplingProfiler()` / `HermesInternal.disableSamplingProfiler()`; save with `HermesInternal.dumpSampledTraceToFile(path)`. Open the output in `chrome://tracing` or Chrome DevTools Performance. Shows JS function samples correlated with timestamps — pairs with Instruments to understand "this JS function was hot while the UI thread was blocked".

## Analyzing JS Thread on iOS

In Time Profiler, filter to the `com.facebook.react.JavaScript` thread. Deep stacks of `JSC`/`Hermes` internals are expected at the top; your code appears below. `requestAnimationFrame`, `setTimeout` callbacks, and React rendering work are all on this thread. JS-thread blocking beyond a frame budget (~16ms) is the universal source of dropped frames from JS.

## Analyzing UI Thread (Main Queue)

The main thread does native view creation, layout, and drawing. RN under the Old Architecture does Yoga layout on a separate shadow thread, then commits on the main thread; under Fabric, commits are queued but still hit the main thread. Long main-thread tasks (file reads, synchronous native module calls on main queue) directly cause jank. Profile main-thread work with Time Profiler + the Main Thread filter.

## Energy Impact and Battery Testing

Energy Log instrument shows CPU, location, networking, and display energy per second. High-energy regions highlight unexpected background work (polling, animations that don't pause when off-screen). Energy tests on device require unplugging — simulator energy data is not representative.

## Memory Graph Debugger

Pause the debugger → Memory Graph button (looks like three dots). Shows the object graph with retain counts; useful for finding retain cycles. Filter by class to see all live instances of a type — 1000 live instances of a screen's view controller means you're leaking screens.

## Disk Activity Profiling

File Activity instrument captures reads, writes, and mmaps. Surprisingly often, app slowness is caused by synchronous disk access on a critical thread. Image caches, logs, and AsyncStorage (when misused) are common culprits.

## Launch Time Measurement

Instruments has an "App Launch" template that captures pre-main + main + first-frame phases. Or set the `DYLD_PRINT_STATISTICS` env var in the scheme to log dyld timing. Launch perf breaks into: dyld (library load), ObjC runtime init, `main()` → `RCTRootView` init → first JS load → first commit. Each phase can be profiled separately.

## Simulator Limitations

The simulator runs x86_64 (Intel) or arm64 (Apple Silicon) — not the device's ARM. CPU is far faster; GPU is desktop-class; disk and network are host-speed. Perf numbers from simulator are directionally correct for UI thread / JS thread patterns, useless for absolute performance. Always validate on device, preferably the lowest-spec one you support.

## Debugging Release-Config Builds

Debug builds have optimizations off, extra tracing on, and `__DEV__` checks live. For representative perf, profile Release builds. Create a "Profile" scheme that uses the Release configuration but keeps debug symbols; that's what `Product → Profile` uses by default. Don't optimize based on Debug numbers — they're a different workload.
