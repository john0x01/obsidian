# C++ Modules

A **C++ TurboModule** lets you implement the non-UI core of a feature once and call it from JavaScript on both iOS and Android. This note covers when to reach for one, the JSI foundation it builds on, the platform bridging required, and the toolchain, memory, threading, and performance concerns involved.

## Why C++ for Cross-Platform Logic

A C++ TurboModule is the right choice when the logic is non-trivial, performance-sensitive, and platform-independent: crypto primitives, SQLite wrappers, video/audio pipelines, ML inference, protocol parsers. Avoid it for simple platform wrappers — the build complexity of NDK + CocoaPods + CMake dwarfs the benefit.

## JSI as the Foundation

The **JavaScript Interface (JSI)** is a C++ API that lets native code call JS and vice versa without a bridge. Every Hermes/JSC runtime exposes a `jsi::Runtime` object, and you can define `HostObject` subclasses whose properties and methods are invocable from JS as ordinary objects. TurboModules are built on `HostObject`. The reasons to reach for JSI directly are synchronous calls, zero-copy arguments (for `ArrayBuffer`), and type-safe dispatch.

## HostObject and HostFunction

`jsi::HostObject` has `get(runtime, name)` and `set(runtime, name, value)`; you dispatch to fields or generate functions lazily. `jsi::Function::createFromHostFunction` wraps a C++ lambda into a JS callable. The lambda receives `(Runtime&, Value thisVal, const Value* args, size_t count)` and returns `jsi::Value`. Exceptions propagate as JS exceptions if you throw `jsi::JSError`; plain C++ exceptions crash the runtime.

## Writing a C++ TurboModule Spec

A TurboModule **spec** is a TypeScript interface extending `TurboModule`. Codegen produces a `NativeMyModuleSpecJSI.h` with a protocol `NativeMyModuleCxxSpec<T>`; your C++ class inherits from it and implements the methods. The spec defines argument and return types — unsupported types (functions with closures, native handles) will not Codegen. Use `Object` or a typed struct for complex returns.

## Bridging Objective-C++ and C++ (iOS)

On iOS, the TurboModule bridge between JS and C++ lives in Objective-C++ (`.mm` files) that conform to the `RCTTurboModule` protocol. Your `.mm` file imports `NativeMyModuleSpecJSI.h` and returns a `std::shared_ptr<facebook::react::TurboModule>` from the spec's factory method. The C++ core itself lives in `.cpp` files built alongside; the `.mm` is thin glue.

## Bridging JNI and C++ (Android)

Android's TurboModule connects Java and C++ via JNI. The generated `NativeMyModuleSpec.java` has a native method bound to your C++ through JNI registration. **fbjni** (Facebook's JNI library) is the idiomatic way to express the JNI surface in C++: it provides RAII over JNI references, exception translation, and type-safe argument marshalling. Raw JNI works but is brittle.

## CMake and NDK Toolchain

Android builds C++ via CMake invoked by Gradle's `externalNativeBuild`. You provide a `CMakeLists.txt` that declares sources, include paths, and link flags. The NDK toolchain cross-compiles for each ABI (`arm64-v8a`, `armeabi-v7a`, `x86_64`). The choice of `APP_STL=c++_shared` versus `c++_static` affects app size and crash behavior; `c++_shared` is the default under modern NDK. Keep the NDK version pinned — mismatches across modules break builds.

## Codegen from TypeScript Specs

RN **Codegen** reads spec files, produces C++ headers and JS shims, and emits them during `pod install` (iOS) and `./gradlew preBuild` (Android). Never edit generated files; regenerate by bumping the spec version. For monorepo setups, point Codegen at the right spec directories via `react-native.config.js`. Diagnostic output is verbose — read it when spec changes do not propagate.

## Memory Ownership (shared_ptr, unique_ptr)

TurboModules are shared-owned (`std::shared_ptr<TurboModule>`): the runtime holds one reference and your registration code the other. Inside the module, prefer `unique_ptr` for owned resources and `shared_ptr` only when necessary. JSI `Runtime&` references are non-owning — never cache a `Runtime*` beyond the JS call scope, as the runtime may be torn down.

## Threading Model (Runtime Thread Affinity)

A JSI `Runtime` is bound to a single thread, typically the JS thread. Method calls happen on that thread; invoking a JSI API from a background thread corrupts state. For async C++ work, dispatch to a worker thread, then post back to the JS thread via the `CallInvoker` (provided to your TurboModule) before touching JSI. Fabric's UI runtime is a separate `Runtime` on the UI thread — do not cross them.

## Synchronous vs Asynchronous Methods

JSI supports synchronous return values: the method blocks JS until it returns. This is ideal for fast operations (hashing, small parses) where the bridge-hop cost of async would dominate. For anything that could take more than 1 ms, return a Promise built from `jsi::Function` + `CallInvoker` to hop to a background thread. Blocking the JS thread for long synchronous calls causes jank just as badly as blocking it in JS.

## Error Propagation Across Language Boundaries

Throw `jsi::JSError(runtime, "message")` to surface a JS-level `Error` with a proper stack. Catch `std::exception` at the TurboModule boundary and translate it to a `JSError` — uncaught C++ exceptions are undefined behavior in most runtimes. For async code, reject the returned Promise with a `JSError`-compatible value. Keep error codes stable; JS code branches on them.

## Third-Party C++ Libraries (Static vs Shared Linking)

**Static linking** embeds the library in your module's `.so`/binary; it is simple, but multiple modules bundling the same library multiply app size and can cause symbol conflicts. **Shared linking** uses a single copy (e.g., OpenSSL from the system or a shared `.so`) — less size, but more platform variability. Document which libraries are static and watch for duplicate-symbol errors at link time.

## Performance Benefits vs Pure Native

A well-designed C++ TurboModule beats a bridge-based native module by 10–100× on call overhead; the actual work is native either way, so the speedup is in the dispatch. Synchronous returns eliminate promise microtask latency entirely. The win is biggest for high-frequency small calls (per-frame computations, gesture handlers, stream decoders); for occasional large operations, Old Architecture modules are still fine.

## Debugging C++ via LLDB / GDB

Xcode attaches LLDB to iOS apps by default — breakpoints in `.mm` and `.cpp` work if symbols are generated (`-g`). Android Studio attaches LLDB to NDK code; use `Debug → Attach Debugger → Dual (Java + Native)`. For release builds, keep unstripped `.so` files (the "debug symbols" toggle in Gradle) to symbolicate crashes. `ndk-stack` translates raw crash dumps from `adb logcat` into function + line.

## ABI Stability Across NDK Versions

C++ has no stable ABI — mixing `.so` files built with different NDK versions or `-std=c++XX` flags is undefined behavior. Pin the NDK version in `build.gradle` (`ndkVersion "26.2.11394342"`) and document it in the README. When a third-party `.aar` includes a prebuilt `.so`, verify NDK compatibility before integrating. iOS is less picky because Xcode handles the toolchain, but C++ standard library mixing is still a hazard.

## Use Cases (Crypto, ML, Parsers)

Good fits: libsodium wrappers, MLKit / Core ML inference loops, Protobuf/FlatBuffers decoders, custom video/audio processing, SQLite/DuckDB wrappers for analytics, and cryptographic signing and verification. Bad fits: platform-API wrappers (use Swift/Kotlin), UI components (use Fabric), and one-off utilities. A useful rule: if you would pick C++ for the equivalent desktop library, it fits here.

## Testing C++ Cores

The C++ core should be testable without RN — link it into a separate CMake test target using GoogleTest or Catch2. Test business logic there, reserving RN-integration tests for the JSI boundary and platform glue. This gives fast, non-UI tests that run on the development machine and catch most bugs before a device build is needed. Coverage instrumentation (`gcov`/`lcov`) works on the standalone target.

## See Also
- [[jsi]] — the C++ API this builds on
- [[turbo-modules]] — C++ modules are TurboModules
- [[native-modules]] — platform-language alternative
- [[codegen]] — generates the spec headers
- [[threading-model]] — runtime thread affinity rules
