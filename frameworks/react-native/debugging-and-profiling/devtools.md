# DevTools

## RN Dev Menu and Its Features

Triggered by shake gesture on device, `Cmd+D` on iOS simulator, `Cmd+M` on Android emulator. Offers Reload, Debug (launches external debugger), Enable Fast Refresh, Toggle Inspector, Show Perf Monitor, and per-build extensions. It only exists in dev builds — shipping it in release is a common footgun (accidental `__DEV__` checks). Some teams expose a hidden dev menu in release via a secret gesture for QA builds; gate it behind a feature flag and build-variant check.

## Chrome DevTools for Hermes (0.76+)

RN 0.76 ships a direct Chrome DevTools integration: launch the app, open `chrome://inspect`, pick the Hermes runtime, get a full DevTools session. Sources with breakpoints, console, Performance tab, Memory tab all work. This replaces the deprecated remote JS debugger and Flipper's JS debugger. Significant improvement — prior debugging was either slow (remote) or limited (Flipper).

## React DevTools

Launched via `npx react-devtools`, connects over WebSocket. The Components tab shows the React tree, props, and state; the Profiler tab records render times per component. "Record why each component rendered" attributes re-renders to specific prop/state/context changes — indispensable for finding identity-equality bugs. Works alongside Chrome DevTools; the two complement each other (React tree vs JS execution).

## React Native DevTools

A newer integrated tool that bundles Chrome DevTools + React DevTools with RN-specific panels (performance overlay, network, bridge inspector for old arch). Launches from the dev menu. Still evolving as of 0.76+; gradually replacing the ecosystem of separate tools.

## Perf Monitor

Dev-menu option overlays live stats: JS FPS, UI FPS, RAM, JS heap. Jank shows as JS FPS dropping below 60 (usually) while UI FPS stays at 60 (rare) — that gap pinpoints whether the JS thread or the UI thread is the bottleneck. Keep it running during manual QA passes; patterns emerge over time.

## Flipper (Deprecated)

Meta's cross-platform debugger. Deprecated as of RN 0.74 due to maintenance cost and architecture misalignment. Still works if you configure it explicitly, but don't invest in new tooling around it. The features that mattered (network inspection, layout inspector, crash reporter) are available elsewhere now.

## Metro Inspector

Metro's web interface (usually at `localhost:8081`) exposes bundle info, module graph, and a log stream. `localhost:8081/debugger-ui` historically hosted the remote debugger (deprecated). `localhost:8081/inspector` offers a module inspector. Most useful for understanding bundle size and unexpected dependency inclusions — the module graph view shows who's importing what.

## Network Inspection

Chrome DevTools Network tab captures fetch/XMLHttpRequest when the JS debugger is attached. For native-level requests (native modules using URLSession / OkHttp directly), use Charles Proxy, mitmproxy, or Proxyman — set the device to route through the proxy and install its cert. Network inspection is where API contract bugs surface; make it part of the on-boarding setup.

## Element Inspector

Dev menu → Toggle Inspector. Tap any view to see its component name, props, styles, and hierarchy. Useful for "which component is rendering this pixel?". Accessibility Inspector on iOS (and Accessibility Scanner on Android) is the a11y equivalent — run it before shipping.

## Log Inspection (logcat, Console app)

Android: `adb logcat`, filter by tag `ReactNativeJS` for JS logs; `*:E` for errors. iOS: macOS Console app, filter by the app's process, or `xcrun simctl spawn booted log stream --level=debug`. `console.log` from JS lands in both places in dev. In release, disable `console.*` calls (Metro's `inline-requires` + a Babel plugin) to avoid shipping noise.

## Redux DevTools Integration

`redux-devtools-extension` wires Redux state to the Chrome extension; `remotedev` works over network for device inspection. Time-travel debugging is genuinely useful for complex state bugs. Modern alternatives (Zustand, Jotai) have lighter dev setups — each has its own devtool hook or browser extension.

## Reactotron

Cross-platform desktop debugger focused on RN: live state inspection, API call logging, benchmark plotting, async storage explorer. Simpler setup than Chrome+React DevTools for small teams. Active maintenance, though its niche is narrowing with Chrome DevTools integration in 0.76+.

## Remote JS Debugger (Deprecated)

Historically, "Debug JS Remotely" from the dev menu shipped JS execution to Chrome — JS ran in the browser, bridged to native. Removed in 0.76. If you see old docs mentioning it, ignore — Chrome DevTools direct Hermes integration is the replacement.

## Crash Reporters (Sentry, Crashlytics, Bugsnag)

Mandatory in production. Configure to capture both JS errors (including unhandled promise rejections) and native crashes (symbolicated via dSYMs / Android mapping files). Set release tags per build so regressions are pinnable to specific versions. Track crash-free rate as a release gate (see build-and-deploy/releasing).

## On-Device Debug Overlays

For field debugging on user devices (with consent), some apps embed an in-app debug overlay showing request logs, feature flag values, device metadata. Useful for remote QA. Gate behind a secret code entry, never ship enabled by default — leaking sensitive info via debug overlays is a real security failure.

## Secure Debug Flag Gating

All of the above should be off in release. Relying on `__DEV__` works but a single `if (someVar)` bypass accidentally committed breaks the invariant. Safer: build-variant separation (dev, staging, prod targets each disable/enable features explicitly) and a pre-release lint that fails on debug flags enabled in prod configs.
