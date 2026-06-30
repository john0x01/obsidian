# New vs Old Architecture

## Old architecture overview

### The Bridge

The bridge was a synchronous message queue across threads. JS would batch up calls into a queue, the queue got serialized to JSON, then drained on a different thread (or vice versa). The actual problem was not the JSON serialization, but instead the queue architecture.

## New architecture overview

### jsi:Runtime

jsi::Runtime is an abstract base class. It has virtual methods like evaluateJavaScript, global, createObject, createStringFromUtf8, etc. JSI itself does not implement these. JSI does not contain a JavaScript engine.

Instead, each engine provides its own subclass:
- Hermes → HermesRuntime : public jsi::Runtime
- JavaScriptCore → JSCRuntime : public jsi::Runtime
- V8 (in some forks) → V8Runtime : public jsi::Runtime

When you call runtime.createStringFromUtf8(...), you're calling a virtual method. At runtime, that dispatches to whichever engine is actually loaded. The engine knows how to turn your const uint8_t* into one of its own internal string representations (a Hermes HermesValue, a JSC JSStringRef, etc.).

### How the native code assembles to JSI

At app boot, React Native constructs a runtime object with your JavaScript engine (Hermes, JSC), a concrete subclass of jsi::Runtime. The native code gets a reference to this runtime and calls into JSI to attach host functions and host objects to the JS global. From that point on, JS code can call those host functions; each call is a direct C++ function invocation on the JS thread, no serialization, no queue.

### Shadow Tree

The Shadow Tree is the C++ representation of your UI as a tree of shadow nodes — one per React host component (<View>, <Text>, etc.). It's the source of truth for layout and is immutable: every update produces a new tree, which is what makes concurrent rendering safe.
Threads and phases:

Render phase — runs on the JS thread. React reconciles, and as it does, Fabric creates the shadow nodes (via Codegen-generated C++ component descriptors). Output: a new Shadow Tree.
Commit phase — runs on a background thread (the "commit"/layout thread, not JS, not main). Two things happen: Yoga lays out the tree, and the new tree is promoted to be the next mounted tree.

Yoga's input: the tree of nodes with their style props (flex, width, padding, etc.).
Yoga's output: a LayoutMetrics per node — absolute frame (x, y, width, height) computed from the flexbox rules. Yoga is pure: styles in, geometry out.


Mount phase — runs on the main/UI thread. Fabric diffs the previous mounted tree against the new one, produces a list of mutations (create view, update props, delete, reparent), and applies them to the real UIView/android.view.View objects.

The mental hook: JS builds it → background thread measures it (Yoga) → main thread paints it (mount). The win over the old arch is that layout no longer requires an async round-trip to JS, and the immutable tree lets React interrupt/throw away in-progress renders without corrupting what's on screen.