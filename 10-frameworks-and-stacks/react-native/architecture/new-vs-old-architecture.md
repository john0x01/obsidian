# New vs Old Architecture

A React Native app is split into two worlds: the **JavaScript** side (your React code) and the **native** side (the platform UI and APIs). The central design question is how those two worlds communicate. The *old* architecture answered it with an asynchronous **Bridge**; the *new* architecture (Fabric + TurboModules, built on **JSI**) replaces the bridge with direct, synchronous calls.

## Old Architecture

### The three threads

An app ran across three threads:

- **JavaScript thread** — runs the JS bundle inside a JavaScript engine (originally JavaScriptCore).
- **Native / UI (main) thread** — runs native code and performs all UI work: rendering, gestures, and event handling.
- **Shadow thread** — computes layout (the position and size of native elements) using Yoga.

### The Bridge

All communication between the JS and native threads went through a single component called the **Bridge**: an asynchronous, batched message queue. Calls were serialized to JSON on one side, queued, and deserialized on the other. The real bottleneck was not the JSON encoding itself but the single-queue, fully asynchronous design.

### Drawbacks

1. **Serialization + single-queue bottleneck.** Every UI update, native-module call, and event crossed one serialized async queue. Under load — animations, large-list scrolling, gestures, rapid state changes — the queue backed up, causing dropped frames and jank and making a steady 60 FPS hard to hold.
2. **Strictly asynchronous (no synchronous calls).** JS could not block and wait for a native result. Synchronous reads — e.g. measuring a view's dimensions for a tooltip or for positioning — were impossible, producing visible glitches: blank list rows, intermediate-state flashes, UI jumps.
3. **Centralized UIManager.** All UI updates flowed through a single `UIManager` that mutated one native view hierarchy on one thread, leaving no room for concurrent or off-main-thread rendering of urgent updates.
4. **Memory overhead and view recreation.** Views were frequently recreated or heavily mutated; data was duplicated (the JS shadow tree plus the native views) and copied again during serialization.
5. **Limited extensibility, hard debugging.** Deep native integration was awkward, the runtime was tied to a specific JS engine, and bridge desynchronisation produced opaque errors. Working around these limits (`InteractionManager`, manual batching, avoiding certain patterns) became common senior-level knowledge.

These structural constraints — not any single bug — are why the architecture was replaced.

## New Architecture

The new architecture removes the bridge. Instead of serializing messages across a queue, JavaScript and C++ talk directly through **JSI**, and a new C++ rendering pipeline (**Fabric**) drives layout and mounting.

### JSI and `jsi::Runtime`

`jsi::Runtime` is an abstract C++ base class with virtual methods such as `evaluateJavaScript`, `global`, `createObject`, and `createStringFromUtf8`. JSI itself contains **no** JavaScript engine — each engine supplies a concrete subclass:

- Hermes → `HermesRuntime : public jsi::Runtime`
- JavaScriptCore → `JSCRuntime : public jsi::Runtime`
- V8 (some forks) → `V8Runtime : public jsi::Runtime`

Calling `runtime.createStringFromUtf8(...)` invokes a virtual method that dispatches to whichever engine is loaded; the engine converts your `const uint8_t*` into its own internal representation (a Hermes `HermesValue`, a JSC `JSStringRef`, etc.).

At boot, React Native constructs the runtime with the chosen engine, hands native code a reference to it, and native code attaches **host functions** and **host objects** to the JS global. From then on, a JS call into native is a **direct C++ function invocation on the JS thread** — no serialization, no queue.

### The Shadow Tree and the render pipeline

The **Shadow Tree** is the C++ representation of the UI: one shadow node per host component (`<View>`, `<Text>`, …). It is the source of truth for layout and is **immutable** — every update produces a new tree, which is what makes concurrent rendering safe (React can interrupt or discard an in-progress render without corrupting the screen).

Rendering proceeds in three phases across three threads:

1. **Render phase (JS thread).** React reconciles; Fabric creates shadow nodes via Codegen-generated C++ component descriptors. Output: a new Shadow Tree.
2. **Commit phase (background layout thread).** Yoga lays out the tree — a pure function: style props (flex, width, padding…) in, `LayoutMetrics` (absolute x/y/width/height) out — and the new tree is promoted to become the next mounted tree.
3. **Mount phase (main/UI thread).** Fabric diffs the previous mounted tree against the new one, produces a mutation list (create view, update props, delete, reparent), and applies it to the real `UIView` / `android.view.View`.

The mental model: **JS builds it → a background thread measures it (Yoga) → the main thread paints it (mount).** The key win over the old architecture is that layout no longer needs an async round-trip to JS, and the immutable tree enables safe concurrent rendering.
