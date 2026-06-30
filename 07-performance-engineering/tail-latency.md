# Tail Latency

## Why Tails Matter (User Experience, Fan-Out Amplification)

## Percentiles (p50, p95, p99, p999)

## Tail Amplification in Fan-Out Services

## The Tail at Scale (Dean & Barroso)

## Sources of Tail Latency

### Garbage Collection as Tail Source

### JIT Compilation Stalls

### Lock Contention as Tail Source

### Noisy Neighbors

## Mitigation Techniques

### Hedged Requests

### Tied Requests

### Backup Requests

### Speculative Execution

### Request Reordering

### Load Balancing to Reduce Tail

## Measuring and Reporting Tails

### Histogram Design for Tails

### Tail Latency SLOs

## See Also
- [[latency-vs-throughput]] — canonical latency context for tails
- [[garbage-collection]] — GC pauses as a tail source
- [[jit-vs-aot]] — JIT stalls as a tail source
- [[06-software-design/system-design/load-balancing|load-balancing (system design)]] — balancing to reduce tail
