# Native Modules

## What a Native Module Is

A native module exposes platform code (Objective-C/Swift on iOS, Java/Kotlin on Android) to JavaScript as callable methods. It's the escape hatch for anything the JS API doesn't cover — filesystem, biometrics, payments, platform SDKs. Under the Old Architecture they communicate via the bridge (async JSON messages); under the New Architecture they're TurboModules, connected via JSI with typed contracts and optional synchronous calls.

## Old Arch: RCTBridgeModule / ReactContextBaseJavaModule

On iOS you conform to `RCTBridgeModule` and register with `RCT_EXPORT_MODULE()`; methods are exposed with `RCT_EXPORT_METHOD`. On Android you extend `ReactContextBaseJavaModule` and annotate methods with `@ReactMethod`. All calls are asynchronous because every invocation serializes through the bridge's JSON queue, even when the native method is instant. The bridge batches and throttles calls, which means no guaranteed call ordering across modules.

## New Arch: TurboModules and Spec Files

TurboModules use JSI to expose methods directly on a JS HostObject — no serialization, no bridge queue, optional synchronous results. You write a TypeScript (or Flow) *spec* that describes the module's interface; Codegen produces C++ headers, Objective-C protocols, and Java interfaces. Native code implements the generated interface; the runtime resolves method calls through a virtual dispatch table. Lazy by default — the module isn't instantiated until first JS access.

## Method Signatures and Supported Types

Primitive types (number, string, boolean), arrays, and objects that map to `NSDictionary`/`ReadableMap` work across both architectures. TurboModules additionally enforce typed argument and return shapes at the Codegen layer. Functions, class instances, and custom types do not cross the boundary — you marshal them as opaque handles or plain data. Nullability is respected under the New Architecture, which surfaces real bugs that the Old Arch silently tolerated.

## Promise-Based Methods

On iOS, the resolver/rejecter signature is `RCTPromiseResolveBlock resolve, RCTPromiseRejectBlock reject`; on Android, `Promise` with `.resolve()` and `.reject()`. Resolving twice is a runtime error on iOS (crash in debug) and a warning/noop on Android; forgetting to resolve keeps the JS promise pending forever and retains the closure. On error paths, always reject with a stable string code and a descriptive message — the JS side parses the code to branch.

## Callback-Based Methods (Legacy)

The older `Callback` signature takes one-or-two `Callback` arguments (success/failure). Callbacks are one-shot; invoking twice is an error. Prefer Promises unless you're maintaining an existing API surface. Under TurboModules, callbacks still work but Promises Codegen cleaner.

## Event Emitters from Native to JS

iOS: subclass `RCTEventEmitter`, declare supported events in `supportedEvents`, emit with `sendEventWithName:body:`. Android: use `DeviceEventManagerModule.RCTDeviceEventEmitter` via `getJSModule`. On the JS side, `NativeEventEmitter` subscribes to the event and returns a subscription you must `.remove()` in cleanup. Emitting before any listener is registered drops the event silently; buffer on native if first-event semantics matter.

## Threading (UI, JS, Background Queues)

By default, RN chooses a queue for your module methods. iOS: override `methodQueue` to force execution on `dispatch_get_main_queue()` for UI work or a private queue for I/O. Android: override `getName` and use `@ReactMethod(isBlockingSynchronousMethod = true)` sparingly — blocking synchronous calls jam the JS thread. For expensive work, dispatch off the default queue yourself and resolve the promise from the background.

## Memory Management in Native Modules

ObjC ARC and Java GC handle most cases, but bridged objects create subtle cycles. On iOS, a module retaining a promise resolver that captures `self` inside a delegate closure is a classic leak. On Android, emitter subscriptions held by `ReactApplicationContext` can keep the module alive past its expected lifetime. Test with Xcode Memory Graph Debugger and Android LeakCanary.

## Error Handling and Reject Paths

Every failure branch must either resolve or reject once — never both, never neither. Common bug: an async native call resolves the promise and then, on an error in a completion handler, also rejects. Establish a convention: wrap the body in a try/catch that rejects at the top level, so exceptional paths converge on a single reject site. Include machine-readable error codes (`"E_NETWORK_TIMEOUT"`) not just messages.

## iOS Implementation (Objective-C / Swift)

Objective-C is the historical path and still the default for most third-party modules; Swift is supported via an Objective-C shim (`@objc(ModuleName)` plus a matching `.m` bridge file). Swift modules are cleaner for concurrency (`async`/`await`, `Task`) but the bridge overhead and import costs are real. For New Arch TurboModules, the generated protocol makes Swift conformance straightforward.

## Android Implementation (Java / Kotlin)

Java is the historical baseline; Kotlin is idiomatic and fully supported. Annotations, lifecycle, and threading are identical — Kotlin is a better language for concurrency (coroutines) but buys no perf difference. For TurboModules, the generated Java interface is consumable from both languages equivalently.

## Registering Modules (Packages and Module Providers)

iOS: `[ReactNativeFactory createRootViewWithBridge:...]` picks up modules registered via `RCT_EXPORT_MODULE`; for extra control, conform to `RCTBridgeDelegate` and return module instances. Android: implement `ReactPackage` with `createNativeModules(...)` returning your list, then register the package in `MainApplication`. TurboModules add a `TurboModuleManagerDelegate` that maps module names to instances.

## Lazy Loading of Modules

Old Arch modules construct at bridge startup unless you implement `requiresMainQueueSetup` (iOS) returning false and defer work to first method call. New Arch TurboModules are lazy by default — the module instance is created on first JS access. Lazy loading matters for startup time; front-loading ten modules that do SDK init adds up fast.

## Autolinking and Distribution

A library's `package.json` has a `react-native.config.js` or `react-native` field that tells the CLI where to find native sources. CocoaPods integrates via a generated Podspec reference; Gradle integrates via a generated `settings.gradle` include. For internal modules, a monorepo workspace alias works; for public packages, publish with sources and let autolinking do the rest.

## Testing Native Modules

Unit-test the native side with XCTest (iOS) and JUnit / Robolectric (Android) — no RN bridge needed. Integration tests that span JS↔native require a running RN instance; Detox or a standalone test harness with a bridge works. Mock the module in Jest (`jest.mock('react-native', () => ({ NativeModules: { MyModule: { ... } } }))`) for pure-JS tests.

## Expo Modules API as Alternative

`expo-modules-core` provides a declarative, Swift/Kotlin-first module API with built-in Codegen, lifecycle hooks, and typed events. Less boilerplate than raw RCT modules, and the same module works in both Expo-managed and bare workflows. For new modules, worth evaluating even if you don't use Expo app-side — the DX win is large.

## Migration from Old Arch to TurboModules

Step 1: write a spec file (`NativeMyModule.ts`) describing the surface. Step 2: run Codegen to produce headers/protocols. Step 3: conform your existing native class to the generated protocol/interface. Step 4: register via the TurboModule manager rather than the legacy package. Many modules support both architectures via compile-time flags (`RCT_NEW_ARCH_ENABLED`). Plan for API tweaks — stricter typing catches latent bugs.

## Common Pitfalls

Resolving a promise twice. Forgetting `sub.remove()` on JS subscriptions. Retaining `self` in an iOS completion block that captures the resolver. Blocking the JS thread via a long synchronous method. Emitting events before any listener exists. Hardcoding main-queue assumptions that break when the JS thread isn't the main thread. All of these are hard to catch in review and easy to find in production.
