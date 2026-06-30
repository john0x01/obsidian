# Garbage Collection

## Reference Counting

## Tracing Collectors

### Mark and Sweep

### Mark and Compact

### Copying Collectors

## Generational Hypothesis and Collectors

## Compaction to Avoid Fragmentation

## Concurrency

### Concurrent vs Stop-the-World

### Incremental GC

### Tri-Color Abstraction

### Write Barriers and Read Barriers

### Card Marking

## Region-Based GC (G1, ZGC, Shenandoah)

## References and Finalization

### Weak, Soft, Phantom References

### Finalizers and Cleaners Pitfalls

## Tuning and Measurement

### GC Tuning Parameters

### Measuring GC Pauses

## Escape Analysis and Stack Allocation

## Non-GC Memory Models (Rust, C++)

## See Also
- [[garbage-collector]] — JS engine GC implementation
- [[memory-bound-optimization]] — reducing GC allocation pressure
- [[tail-latency]] — GC pauses as a tail source
- [[03-computer-systems/operating-systems/memory-management|memory-management (OS)]] — OS memory model context
