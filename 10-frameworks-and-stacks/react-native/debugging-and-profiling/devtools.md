# DevTools

## RN Dev Menu and Its Features

The **dev menu** is triggered by a shake gesture on a device, `Cmd+D` on the iOS simulator, or `Cmd+M` on the Android emulator. It offers Reload, Debug (launches an external debugger), Enable Fast Refresh, Toggle Inspector, Show Perf Monitor, and per-build extensions. It exists only in dev builds; shipping it in release is a common footgun caused by accidental `__DEV__` checks. Some teams expose a hidden dev menu in release via a secret gesture for QA builds — gate it behind a feature flag and a build-variant check.

## JS Debugging

### Chrome DevTools for Hermes (0.76+)

RN 0.76 ships a direct **Chrome DevTools** integration: launch the app, open `chrome://inspect`, and pick the Hermes runtime to get a full DevTools session. Sources with breakpoints, the console, the Performance tab, and the Memory tab all work. This replaces the deprecated remote JS debugger and Flipper's JS debugger — a significant improvement, since prior debugging was either slow (remote) or limited (Flipper).

### React DevTools

**React DevTools**, launched via `npx react-devtools`, connects over WebSocket. The Components tab shows the React tree, props, and state; the Profiler tab records render times per component. "Record why each component rendered" attributes re-renders to specific prop/state/context changes, making it indispensable for finding identity-equality bugs. It works alongside Chrome DevTools, the two complementing each other (React tree vs. JS execution).

### React Native DevTools

**React Native DevTools** is a newer integrated tool that bundles Chrome DevTools and React DevTools with RN-specific panels (performance overlay, network, and a bridge inspector for the old architecture). It launches from the dev menu. Still evolving as of 0.76+, it is gradually replacing the ecosystem of separate tools.

### Remote JS Debugger (Deprecated)

Historically, "Debug JS Remotely" in the dev menu shipped JS execution to Chrome — JS ran in the browser and bridged to native. It was removed in 0.76. If old docs mention it, ignore them; the direct Chrome DevTools Hermes integration is the replacement.

## Inspection Tools

### Perf Monitor

**Perf Monitor**, a dev-menu option, overlays live stats: JS FPS, UI FPS, RAM, and JS heap. Jank usually shows as JS FPS dropping below 60 while UI FPS stays at 60; the rarer reverse also occurs. That gap pinpoints whether the JS thread or the UI thread is the bottleneck. Keep it running during manual QA passes so patterns emerge over time.

### Element Inspector

The **element inspector** (dev menu → Toggle Inspector) lets you tap any view to see its component name, props, styles, and hierarchy — useful for answering "which component is rendering this pixel?". The Accessibility Inspector on iOS (and Accessibility Scanner on Android) is the a11y equivalent; run it before shipping.

### Metro Inspector

The **Metro inspector** is Metro's web interface, usually at `localhost:8081`, exposing bundle info, the module graph, and a log stream. `localhost:8081/debugger-ui` historically hosted the (now deprecated) remote debugger, while `localhost:8081/inspector` offers a module inspector. It is most useful for understanding bundle size and unexpected dependency inclusions — the module graph view shows who is importing what.

### Network Inspection

The Chrome DevTools **Network** tab captures `fetch`/`XMLHttpRequest` when the JS debugger is attached. For native-level requests (native modules using `URLSession`/OkHttp directly), use Charles Proxy, mitmproxy, or Proxyman: route the device through the proxy and install its cert. Network inspection is where API contract bugs surface, so make it part of the onboarding setup.

### Log Inspection (logcat, Console app)

- **Android**: `adb logcat`, filtered by the `ReactNativeJS` tag for JS logs or `*:E` for errors.
- **iOS**: the macOS Console app filtered by the app's process, or `xcrun simctl spawn booted log stream --level=debug`.

`console.log` from JS lands in both places in dev. In release, disable `console.*` calls (via Metro's `inline-requires` plus a Babel plugin) to avoid shipping noise.

## Third-Party and State Debuggers

### Redux DevTools Integration

The **`redux-devtools-extension`** wires Redux state to the Chrome extension, and `remotedev` works over the network for device inspection. Time-travel debugging is genuinely useful for complex state bugs. Modern alternatives such as Zustand and Jotai have lighter dev setups, each with its own devtool hook or browser extension.

### Reactotron

**Reactotron** is a cross-platform desktop debugger focused on RN: live state inspection, API call logging, benchmark plotting, and an async storage explorer. It has a simpler setup than Chrome plus React DevTools for small teams and remains actively maintained, though its niche is narrowing with the Chrome DevTools integration in 0.76+.

### Flipper (Deprecated)

**Flipper** is Meta's cross-platform debugger, deprecated as of RN 0.74 due to maintenance cost and architecture misalignment. It still works if configured explicitly, but do not invest in new tooling around it. The features that mattered — network inspection, layout inspector, crash reporter — are now available elsewhere.

## Production Debugging and Security

### Crash Reporters (Sentry, Crashlytics, Bugsnag)

**Crash reporters** are mandatory in production. Configure them to capture both JS errors (including unhandled promise rejections) and native crashes (symbolicated via dSYMs / Android mapping files). Set release tags per build so regressions are pinnable to specific versions. Track crash-free rate as a release gate (see build-and-deploy/releasing).

### On-Device Debug Overlays

For field debugging on user devices (with consent), some apps embed an **in-app debug overlay** showing request logs, feature-flag values, and device metadata. This is useful for remote QA. Gate it behind a secret code entry and never ship it enabled by default — leaking sensitive info via debug overlays is a real security failure.

### Secure Debug Flag Gating

All of the above should be off in release. Relying on `__DEV__` works, but a single `if (someVar)` bypass committed by accident breaks the invariant. Safer is **build-variant separation** (dev, staging, and prod targets each explicitly enable or disable features) plus a pre-release lint that fails on debug flags enabled in prod configs.

## See Also
- [[ios-profiling]] — deep iOS profiling
- [[android-profiling]] — deep Android profiling
