# JIT vs AOT Compilation

## Ahead-of-Time Compilation

## Just-in-Time Compilation

## Tiered Compilation

## Adaptive Optimization

### Profile-Guided Optimization (PGO)

### Inlining and Escape Analysis

### Deoptimization and Speculation

## Trade-offs (Peak Performance vs Startup)

### Warm-Up Costs in JIT

### Startup Time Penalties

### AOT for Serverless Cold Starts

## Implementations

### HotSpot C1 and C2

### V8 TurboFan, Ignition, Sparkplug

### LLVM-Based JITs

### GraalVM Native Image

## Tiered Storage of Compiled Code

## JIT Security Risks (W^X, Spectre)

## See Also
- [[compilation-optimization]] — the optimizations compilers apply
- [[tail-latency]] — JIT warm-up stalls as a tail source
- [[benchmarking]] — JIT warm-up skews measurements
- [[garbage-collector]] — managed runtimes pair JIT with GC
