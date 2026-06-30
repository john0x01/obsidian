# New Architecture Internals


## iOS Bootstrap chain

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
6. ──── At some point in (4)→(5), there is a hook ────
  ──── for native modules to install JSI bindings ────
      │
      │  This where the install() function runs.
      │  React Native calls it with:
      │     • jsi::Runtime&   (mandatory)
      │     • CallInvoker     (almost always)
      │
      ▼
7. The install() does:
  runtime.global().setProperty(runtime, "storage", ...)
      │
      ▼
8. JS bundle starts executing. globalThis.storage exists.


## Concepts

JS engine operations are only safe on the JS thread