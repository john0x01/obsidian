# Android Profiling

## Android Studio Profiler Overview

The **Android Studio Profiler** (View → Tool Windows → Profiler) attaches to a running app and exposes four top-level tracks: CPU, Memory, Network, and Energy. Each expands into detailed recording modes. Profile *release* builds for numbers that reflect the real user experience — debug builds are substantially slower. To do so, either set `debuggable=true` temporarily or use a `profileable` variant on API 29+.

## CPU Profiling

### CPU Profiler Modes

The CPU Profiler offers three recording modes:

- **Sample Java Methods** — low-overhead statistical sampling across the managed runtime.
- **Trace Java Methods** — records every method entry and exit; exact but high overhead (slows the app ~10×, distorting timing).
- **System Trace** — kernel-level trace via Perfetto; best for I/O, lock, and frame analysis.

For React Native, System Trace is usually the right starting point: it shows the UI thread, JS thread, native threads, and Choreographer frames together.

Use sampling for performance questions and tracing for coverage/call-graph questions. For RN, tracing across the JSI boundary requires enabling both the JS-side Hermes profiler and the Java-side tracer, then correlating timestamps.

### systrace and Perfetto

**`systrace`** is the legacy capture tool; **`perfetto`** replaces it, exposing the same data through a better UI at `ui.perfetto.dev`. Capture from the command line:

```bash
python3 $ANDROID_HOME/platform-tools/systrace/systrace.py --app=com.example.app sched gfx view
```

Open the result in the Perfetto UI for timelines of threads, frames, and custom events. Add custom traces in code via `Trace.beginSection("name")` / `Trace.endSection()`.

## Memory Profiling

### Memory Profiler

The **Memory Profiler** offers two workflows. "Record allocations" tracks every allocation during a period (heavy overhead). "Dump heap" captures a snapshot you can filter by class, sort by retained size, and inspect for references. Nodes pinned by "GC roots" are the actual leak anchors. Compare two heap dumps to find growth.

### LeakCanary for Memory Leaks

**LeakCanary** is a library that automatically detects activities and fragments retained after destruction, then dumps the heap with the retention chain explained in plain English ("this Activity is retained because X, which is held by Y, which is held by Z"). Include it in debug builds only — never in release. It finds roughly 80% of leaks with zero manual profiling.

## Network Profiler

The **Network Profiler** captures `HttpURLConnection`, OkHttp (when using the default interceptor), and platform sockets. It may miss traffic from libraries that use custom TLS or low-level sockets; fall back to Charles or mitmproxy in those cases. The timing breakdown (DNS, connect, TLS, wait, receive) pinpoints whether slowness originates on the backend or the client.

## Energy Profiler

The **Energy Profiler** estimates energy use from CPU, network, location, and wake-lock activity. It is less precise than hardware profiling but good enough to spot obvious offenders such as infinite polling, location updates left on, or animations that never pause. High energy use without user interaction is a red flag.

## Rendering and Frame Analysis

### Layout Inspector

The **Layout Inspector** shows the runtime view hierarchy with properties. Its "3D view" reveals nesting depth, a common source of measurement cost. In RN specifically, excessive nesting often comes from unnecessary `<View>` wrappers; flatten where possible. The inspector updates live as the app runs, so you can step through states.

### GPU Rendering Profiler

Enabling on-device rendering bars via `adb` gives a quick visual check without a laptop profiler:

```bash
adb shell setprop debug.hwui.profile true && adb shell stop && adb shell start
```

Each bar is a frame, color-coded by phase (input handling, animation, measure/layout, draw, execute). Bars exceeding the 16.67 ms horizontal line indicate dropped frames.

### dumpsys gfxinfo for Frame Stats

The following returns the last ~120 frames with per-phase timings (vsync, input handling, measure, draw, sync, command issue, GPU):

```bash
adb shell dumpsys gfxinfo <pkg> framestats
```

Parse this output programmatically for regression detection. A common CI pattern is "run script → swipe feed → read `gfxinfo` → fail if p99 > 30 ms".

### Choreographer and Frame Callbacks

Android's **Choreographer** schedules frame callbacks at VSync and logs "Skipped N frames!" to logcat when the main thread misses its deadline. Watching `logcat | grep Choreographer` during manual testing surfaces frame drops. RN's Fabric mount phase and native view operations run in Choreographer callbacks, so overloading them causes visible jank.

### Android GPU Inspector (AGI)

The **Android GPU Inspector** is a graphical tool for deep GPU performance investigation, offering frame-by-frame analysis of draw calls, shader costs, and texture uploads. It is overkill for most RN work — reach for it only when using OpenGL/Vulkan components (video players, Skia, games) that need per-draw-call detail.

## Diagnosing Main-Thread Problems

### ANR Analysis

An **ANR** ("Application Not Responding") fires when the main thread is blocked for more than 5 seconds. Android captures a trace dump in `/data/anr/` and surfaces it to Play Console → Android Vitals. The dump shows every thread's stack at the ANR moment; find the main thread to see what was blocking. A common RN cause is a synchronous native module call on the main thread that waited on JS.

### StrictMode for Thread Violations

**`StrictMode`** detects violations such as disk I/O on the main thread, network on the main thread, and leaked closeables. Enable it in development via `MainApplication.onCreate`. It can either crash or log on each violation — run the crash mode for a day to surface everything, then relax to logging. Several RN native modules historically violated StrictMode; most are fixed, but third-party libraries may still offend.

## Startup and Method Profiling

### Macrobenchmark for Startup

**AndroidX Macrobenchmark** runs automated startup scenarios on an attached device and reports cold/warm/hot start times with statistical rigor. It integrates with CI for startup-perf regression detection. Generating a baseline profile from Macrobenchmark traces is the fastest path to shaving 20–40% off cold start.

### Method Tracing

**Method tracing** inserts per-method entry/exit logging. "Sample Java Methods" in the profiler is statistical, while "Trace Java Methods" is exact but slows the app ~10× and distorts timing. Use sampling for performance and tracing for coverage/call-graph questions.

## Debugging Release Builds on Android

To profile a **release build**, add `android:debuggable="true"` to the release manifest temporarily (never ship this) or use a `profileable` variant (API 29+, production-safe). Symbolicate native crashes with `ndk-stack` or Play Console's automatic symbolication. JS symbolication needs source maps uploaded to the crash reporter — verify they are landing post-CI.

## See Also
- [[ios-profiling]] — iOS counterpart
- [[devtools]] — general debugging tooling
- [[startup-and-tti]] — profile cold-start
- [[10-frameworks-and-stacks/react-native/performance/memory-management|Memory Management (RN)]] — profile memory growth
