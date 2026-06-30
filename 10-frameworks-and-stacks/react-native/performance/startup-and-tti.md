# Startup and Time to Interactive

## Fundamentals

### Cold Start vs Warm Start vs Hot Start

### Time to Interactive (TTI) Definition

### App Launch Phases (Native Init, RN Init, JS Load, Bundle Parse, First Render)

### First Screen vs Full Hydration Milestones

## Optimization Techniques

### Splash Screen Strategy

### Inline Requires

### RAM Bundles (Old Architecture)

### Hermes Precompiled Bytecode

### Native Module Lazy Initialization

### Deferred Navigation Stack Setup

### Background Preloading Patterns

## Measurement

### Measuring Launch Time (Xcode Instruments, Android Macrobenchmark)

## Platform and Device Concerns

### Impact of App Size on Startup

### Cold Start Cost on Low-End Android

## Diagnostics

### Startup Crash Reporting and Triage

## See Also
- [[bundle-size]] — smaller bundles start faster
- [[metro-bundler]] — bundler controls startup payload
- [[ios-profiling]] — profile cold-start timings
- [[memory-management]] — startup allocations affect TTI
