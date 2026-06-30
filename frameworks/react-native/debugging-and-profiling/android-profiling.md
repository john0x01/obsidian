# Android Profiling

## Android Studio Profiler Overview

Android Studio's Profiler (View → Tool Windows → Profiler) attaches to a running app. Top-level tracks: CPU, Memory, Network, Energy. Each expands into detailed recording modes. Profile *release* builds (with `debuggable=true` temporarily, or a `profileable` variant on API 29+) for numbers that reflect user experience — debug builds are substantially slower.

## CPU Profiler (Sample Java Methods, System Trace)

Three modes: **Sample Java Methods** (low-overhead sampling across the managed runtime), **Trace Java Methods** (every method entry/exit — high overhead), **System Trace** (kernel-level trace via perfetto; best for I/O, lock, and frame analysis). For RN, System Trace is usually the right starting point — it shows the UI thread, JS thread, native threads, and Choreographer frames together.

## Memory Profiler (Heap Dumps, Allocation Tracking)

"Record allocations" tracks every allocation during a period; heavy overhead. "Dump heap" captures a snapshot — filter by class, sort by retained size, inspect references. Nodes marked with "GC roots" pinned in memory are the actual leak anchors. Compare two heap dumps to find growth.

## Network Profiler

Captures HttpURLConnection, OkHttp (when using the default interceptor), and platform sockets. For libraries that use custom TLS or low-level sockets, it may miss traffic — use Charles/mitmproxy as a fallback. Timing breakdown (DNS, connect, TLS, wait, receive) pinpoints whether slowness is backend or client.

## Energy Profiler

Estimates energy use based on CPU, network, location, and wake locks. Not as precise as hardware profiling but good enough to spot obvious offenders (infinite polling, location updates left on, animations not paused). High energy without user interaction is a red flag.

## Layout Inspector

Runtime view hierarchy with properties. "3D view" shows nesting depth — deep hierarchies are common sources of measurement cost. For RN specifically, excessive nesting often comes from unnecessary `<View>` wrappers; flatten where possible. Live updates as the app runs, so you can step through states.

## GPU Rendering Profiler (adb)

`adb shell setprop debug.hwui.profile true && adb shell stop && adb shell start` enables on-device rendering bars. Each bar is a frame, color-coded by phase (input handling, animation, measure/layout, draw, execute). Bars exceeding the 16.67ms horizontal line are dropped frames. Quick visual check without a laptop profiler.

## systrace and perfetto

`systrace` is the legacy tool; `perfetto` replaces it (same data, better UI, at `ui.perfetto.dev`). Capture from command line: `python3 $ANDROID_HOME/platform-tools/systrace/systrace.py --app=com.example.app sched gfx view`. Open in the perfetto UI for timelines of threads, frames, and custom events. Add custom traces via `Trace.beginSection("name") / Trace.endSection()`.

## dumpsys gfxinfo for Frame Stats

`adb shell dumpsys gfxinfo <pkg> framestats` returns the last ~120 frames with per-phase timings (vsync, input handling, measure, draw, sync, command issue, gpu). Parse this programmatically for regression detection — a common CI pattern is "run script → swipe feed → read gfxinfo → fail if p99 > 30ms".

## Choreographer and Frame Callbacks

Android's Choreographer schedules frame callbacks at VSync; it logs "Skipped N frames!" in logcat when the main thread misses its deadline. `logcat | grep Choreographer` during manual testing surfaces frame drops. RN's fabric mount phase and native view operations run in Choreographer callbacks — overloading them causes visible jank.

## Android GPU Inspector (AGI)

Graphical debugging tool for deep GPU perf investigation. Frame-by-frame analysis of draw calls, shader costs, texture uploads. Overkill for most RN work — relevant when you're using OpenGL / Vulkan components (video players, Skia, games) and need per-draw-call detail.

## ANR Analysis

ANR ("Application Not Responding") fires when the main thread is blocked >5 seconds. Android captures a trace dump in `/data/anr/` and surfaces it to Play Console → Android Vitals. The dump shows all thread stacks at the ANR moment — find the main thread, see what was blocking. Common RN cause: synchronous native module call on main thread that waited on JS.

## StrictMode for Thread Violations

`StrictMode` detects violations (disk I/O on main thread, network on main thread, leaked closeables). Enable in dev via `MainApplication.onCreate`. Crashes or logs per violation — use the crash mode for a day, find everything, then relax to logging. Several RN native modules historically violated StrictMode; they've mostly been fixed, but third-party libraries may still.

## LeakCanary for Memory Leaks

Library that automatically detects retained activities/fragments post-destroy and dumps the heap with the retention chain explained in English ("this Activity is retained because X, which is held by Y, which is held by Z"). Include in debug builds; absolutely do not include in release. Finds 80% of leaks with zero manual profiling.

## Macrobenchmark for Startup

Androidx Macrobenchmark runs automated startup scenarios on an attached device and reports cold/warm/hot start times with statistical rigor. Integrates with CI — regression detection for startup perf. Generating a baseline profile from Macrobenchmark traces is the fastest path to the "shave 20–40% off cold start" goal.

## Method Tracing

Inserts per-method entry/exit logging. "Sample Java Methods" in the profiler is statistical; "Trace Java Methods" is exact but slows the app 10× and distorts timing. Use sampling for perf, tracing for coverage/call-graph questions. For RN, tracing across the JSI boundary requires enabling both the JS-side Hermes profiler and the Java-side tracer, then correlating timestamps.

## Debugging Release Builds on Android

Add `android:debuggable="true"` temporarily to the release manifest (don't ship this) or use a `profileable` variant (API 29+, production-safe). Symbolicate native crashes with `ndk-stack` or Play Console's automatic symbolication. JS symbolication needs source maps uploaded to the crash reporter — check they're landing post-CI.
