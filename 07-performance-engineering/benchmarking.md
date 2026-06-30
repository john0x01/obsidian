# Benchmarking

## Microbenchmarks vs Macrobenchmarks

## Realistic Workloads vs Synthetic

## Tooling (JMH, Criterion, Google Benchmark, go test -bench)

## Measurement Methodology

### Warm-Up Phases

### Steady-State Measurement

### JIT Warm-Up in Managed Runtimes

### Cold Cache vs Warm Cache Benchmarks

## Common Pitfalls

### Dead Code Elimination Pitfalls

### Comparing Implementations Fairly

## Environment Control

### CPU Frequency Scaling Effects

### Pinning to Cores

### Benchmark Noise Reduction

## Statistical Analysis

### Benchmark Statistical Analysis

### Variance and Confidence Intervals

## Benchmark Reporting

## Continuous Benchmarking

### A/B Benchmark Infrastructure

### Regression Detection in CI

## See Also
- [[profiling]] — locate hotspots before benchmarking them
- [[big-o-in-practice]] — benchmarking to validate complexity
- [[jit-vs-aot]] — JIT warm-up affects measurements
- [[latency-vs-throughput]] — metrics under measurement
