# Batching

## Why Batching Improves Throughput

## Latency vs Throughput Trade-off

## Batching Strategies

### Fixed-Size vs Dynamic Batches

### Time-Based vs Count-Based Batching

### Adaptive Batching

### Micro-Batching in Stream Processing

## Batch Sizing Heuristics

## Flushing on Backpressure

## Error Isolation in Batches (All-or-Nothing vs Partial)

## Application Domains

### Database Batch Inserts

### Network Batching (Syscall Reduction)

### Protocol-Level Batching (HTTP/2, gRPC)

### Batching in Message Queues

### Batched Cryptographic Operations

### Vectorized Execution (Columnar Databases)

### Batching in GPU Workloads

## Monitoring Batch Effectiveness

## See Also
- [[latency-vs-throughput]] — the core batching trade-off
- [[io-bound-optimization]] — batching to reduce syscalls/I/O
- [[database-performance]] — batch inserts and bulk loads
- [[network-performance]] — request coalescing and batching
