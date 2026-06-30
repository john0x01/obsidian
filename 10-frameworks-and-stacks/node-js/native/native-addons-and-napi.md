# Native Addons And Node-API

A native addon is a dynamically-loaded C/C++ (or Rust) library that Node loads like a module and that runs *inside the process* with direct access to V8 objects, system libraries, and the CPU without a JS interpreter in the way. You reach for one when pure JS hits a wall: a CPU hot path that needs SIMD or hand-tuned C, a system facility with no JS binding (a kernel API, a hardware device, an existing C library you must not rewrite), or interop with a vendor SDK. The central tension is that crossing the JS↔native boundary is *powerful but costly and fragile*, and the history of the API is a long fight against that fragility.

## The ABI Stability Problem (V8/NAN → Node-API)

The original way to write addons was to program **directly against V8's C++ API** (`v8::Local`, `Isolate`, `HandleScope`). V8's C++ API is not stable — its types and signatures change between major versions. So an addon compiled for Node 12's V8 would fail to load on Node 14: you had to recompile, often edit code, for every Node major. **NAN** ("Native Abstractions for Node.js") was a header-only shim that papered over the differences with macros, but you still recompiled per Node version and still ultimately depended on V8 internals.

**Node-API** (historically "N-API") is the durable fix. It is a **C** API — a stable, versioned ABI that Node guarantees across major versions. The contract: an addon compiled against Node-API version *N* keeps loading on every future Node that supports version *N*, with **no recompilation**. It achieves this by hiding V8 entirely behind opaque handle types (`napi_value`, `napi_env`) and a function table the runtime supplies at load time. The engine could even be swapped (Node-API works on non-V8 runtimes) and your addon would not care. `node-addon-api` is the ergonomic **C++** wrapper over the C API — RAII, exceptions, real classes — and is what most people write today; it compiles down to plain Node-API calls.

```
Your addon (C++)
   │  node-addon-api  (C++ sugar, header-only)
   ▼
Node-API            ← STABLE C ABI, versioned, cross-major
   ▼
V8 / engine internals  (volatile — hidden from you)
```

The practical payoff is enormous for the ecosystem: prebuilt binaries actually work. A package can ship a `.node` binary built once and have it load across the whole supported Node range, instead of forcing a compile (and a working toolchain) on every user's install.

## Building: node-gyp and Friends

`node-gyp` is the canonical build tool — a wrapper around GYP/Ninja/Make that compiles `binding.gyp` (the build manifest) against the Node headers for the target version and emits a `.node` shared object. Its reputation for install pain is real: it needs a C/C++ toolchain (Python, plus Visual Studio Build Tools on Windows, Xcode CLT on macOS) on the *end user's* machine if no prebuilt binary is available. The ecosystem mitigates this with **prebuild tooling** (`prebuildify`, `node-pre-gyp`, etc.) that ships compiled binaries per platform/arch so most installs never invoke a compiler — which is only viable *because* Node-API gives ABI stability. `loadablerequire`-ing a `.node` file just `dlopen`s it.

## The Boundary Cost

Every call from JS into native code and back is not free:

- **Marshalling.** JS values must be read out of / written into V8 via Node-API calls (`napi_get_value_double`, `napi_create_string_utf8`, …). Each is a function call across the boundary; touching many small values in a tight loop is where naive addons *lose* to plain JS, because the JIT would have inlined the JS version.
- **Handle scopes and GC interplay.** Native code holds `napi_value` handles the GC must track; mismanaging scopes leaks or, worse, lets V8 move/collect an object you still reference.
- **Threading.** A native thread cannot just touch JS — JS execution is pinned to its Isolate's thread. To call back into JS from another thread you use a **thread-safe function** (`napi_threadsafe_function`), which queues the call onto the JS thread's loop. Blocking native work belongs on libuv's threadpool (`napi_create_async_work`) so it does not stall the event loop.
- **Crash blast radius.** A segfault or a `std::terminate` in the addon takes down the *whole process* — no try/catch saves you. Native bugs are memory-safety bugs.

The design rule that falls out: **make boundary crossings coarse, not fine.** Hand a whole buffer to native code, do the heavy work in one call, return one result. An addon that crosses the boundary per element is usually a regression.

## WASM as the Alternative

WebAssembly is increasingly the answer to "I need native speed but not native danger." You compile C/C++/Rust to a `.wasm` module and run it in V8's WASM engine. Trade-offs versus a native addon:

- **Portability & safety.** One `.wasm` runs on every platform/arch with *no compiler at install time* and executes in a memory-sandboxed VM — a bug corrupts only its linear memory, not the process. This alone wins for untrusted or widely-distributed code.
- **No syscalls by default.** WASM cannot touch the OS directly; it needs a host import layer (WASI gives a standardized one). So for "call a system library / device," a native addon still wins.
- **Performance.** WASM is fast but not always *as* fast as a tuned native addon, and the JS↔WASM boundary plus copying into linear memory has its own cost. For pure compute (codecs, crypto, parsers) it is often close enough and worth the safety.

## Misconceptions and Senior Pitfalls

- **"Native is always faster."** Only past the boundary-cost break-even. Profile first; V8's JIT is excellent, and a poorly-batched addon loses.
- **"Node-API means never recompile."** It means *no recompile across Node majors for the same Node-API version* — you still build per OS/arch, and adopting newer Node-API features raises your minimum version.
- **Writing against raw V8 in 2025.** Almost never justified; it re-buys the per-version recompile pain Node-API exists to kill.
- **Forgetting the threadpool.** Synchronous heavy work in an addon blocks the loop exactly like synchronous JS would — going native does not exempt you from the event-loop contract.

## See Also

- [[worker-threads]]
- [[01-programming-foundations/languages/javascript/engine-internals/v8-architecture|V8 Architecture]]
- [[07-performance-engineering/cpu-bound-optimization|CPU-Bound Optimization]]
- [[07-performance-engineering/jit-vs-aot|JIT vs AOT]]
- [[03-computer-systems/operating-systems/syscalls|System Calls]]
