# New Architecture Internals

## iOS Bootstrap Chain

The sequence below traces how a React Native app starts on iOS, from process launch to the point where native modules have installed their JSI bindings and the JS bundle begins executing.

```text
1. iOS launches your app
      │
      ▼
2. AppDelegate.mm runs (this is your Objective-C++ entry point)
      │
      │  AppDelegate constructs React Native's "host" object:
      │   • Old arch: RCTBridge
      │   • New arch: RCTHost / RCTRootViewFactory
      │
      ▼
3. React Native constructs the JS executor
   (a HermesExecutorFactory or JSCExecutorFactory)
      │
      ▼
4. The executor constructs the concrete jsi::Runtime
   (a HermesRuntime instance, allocated on the heap,
   owned by the executor, accessed via a reference)
      │
      ▼
5. React Native loads the JS bundle (index.jsbundle)
      │
      ▼
6. ──── At some point between (4) and (5), there is a hook ────
   ──── for native modules to install JSI bindings        ────
      │
      │  This is where the install() function runs.
      │  React Native calls it with:
      │     • jsi::Runtime&   (mandatory)
      │     • CallInvoker     (almost always)
      │
      ▼
7. install() does:
   runtime.global().setProperty(runtime, "storage", ...)
      │
      ▼
8. JS bundle starts executing. globalThis.storage exists.
```

## Concepts

- JS engine operations are only safe on the JS thread.

## See Also
- [[new-vs-old-architecture]] — canonical architecture overview
- [[jsi]] — the install() binding mechanism
- [[hermes]] — the HermesRuntime constructed at boot
- [[turbo-modules]] — what installs JSI bindings here
- [[threading-model]] — why JSI is JS-thread-only
- [[10-frameworks-and-stacks/react-native/review-questions|RN Review Questions]] — self-test on architecture
