# React Native Internals (New Architecture)

## 1. Old Architecture

In a nutshell, each react native app is composed of two main parts: The JavaScript code an the native code.

The code is executed over three threads:

- The JavaScript thread: to run JS Bundle with specific JavaScript engine 
- The Native/UI thread: it runs the native code and handles any user interface operation like rendering or gesture events.
- The Shadow thread: which calculates the position of native elements in the layout.

The relationship between the JavaScript and Native threads is mediated by a component called the Bridge.

### Old Architecture drawbacks

Although React Native apps are fast enough, they still lag behind native solutions in terms of performance. The reason lies with the architecture used.

The bridge in the old architecture works by serializing all data that must be passed from JavaScript to native.

Main limitations:

1. Performance Bottleneck from the Single Asynchronous Bridge

- All communication between JS and native went through one serialized, async message queue (the Bridge).
- Every UI update, native module call, event, etc., had to be JSON-serialized on one side and deserialized on the other.
- This introduced significant overhead (latency, CPU, memory) for frequent operations like animations, scrolling large lists, gestures, or rapid state changes.
- Under load, everything queued up resulted in dropped frames, jank, difficulty maintaining 60 FPS.

2. Strictly Asynchronous Nature (No Synchronous Calls)

JS could not block/wait for native responses. This was "non-blocking" in theory but caused real problems:

- No synchronous layout measurements or reads (e.g., getting a view's dimensions in a layout effect for tooltips, positioning, etc.).
- Synchronization issues between JS and native layers → visible glitches like blank list rows, intermediate state flashes, or UI jumps.

Made certain interactions (gestures, precise timing) harder or impossible to implement smoothly.

Centralized UIManager and View Mutation Model
All UI updates went through a single UIManager that mutated a single native view hierarchy.
Layout calculations were constrained to one thread.
No easy concurrent rendering or off-main-thread layout processing for urgent updates (e.g., user input).

Inefficient Memory Usage and View Recreation
Views were often recreated or heavily mutated on updates.
Higher memory footprint due to duplicated data (JS shadow tree + native views) and serialization copies.

Limited Scalability and Extensibility
Hard to integrate deeply with native (e.g., custom complex views, heavy native modules).
Tied to specific JS engine (originally JSC).
Batch processing and message queuing could lead to unpredictable latency.
Difficult to achieve web-like features or advanced concurrent patterns.

Developer Experience and Debugging Pain
Opaque errors when things got out of sync.
Bridge-related issues were notoriously hard to debug (especially in complex apps with many third-party native modules).
Many workarounds (e.g., InteractionManager, manual batching, avoiding certain patterns) became common senior-level knowledge.

At its core, everything between JavaScript and native code had to go through a single asynchronous bridge. Every UI update, event, or native module call was serialized to JSON, sent across the bridge, and deserialized on the other side. This created a major performance bottleneck — especially for animations, scrolling, gestures, and frequent updates. Under pressure, the queue would back up, causing dropped frames and janky experiences.
Another big pain point was that all communication was strictly asynchronous. You couldn’t make synchronous calls from JS to native (for example, measuring a view’s size right when you needed it). This led to timing issues, layout glitches, blank rows in lists, and awkward workarounds.
The architecture also relied on a centralized UIManager that mutated a single view hierarchy. This made it hard to do concurrent work or handle complex native integrations efficiently. On top of that, memory usage was higher because you had duplicated data structures (the JS shadow tree + native views) plus all the copying during serialization.
Debugging bridge-related problems was notoriously frustrating, especially in large apps with many native modules. Seniors often spent time fighting these constraints with tricks like InteractionManager, careful batching, or avoiding certain patterns altogether.
In short, while the old architecture was a clever solution for its time, it became the main limiting factor for high-performance, smooth UIs — which is exactly why the New Architecture (Fabric + TurboModules + JSI) was built to replace it.

## 2. New Architecture Overview

## 3. JSI

## 4. YOGA

## 5. Data flow

## 6. New renderer

## 7. Fabric

## 8. Hermes

## 9. Codegen

## 10. Turbo Modules