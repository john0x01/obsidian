# Atomic Operations

## Atomicity Definition

## Hardware Atomic Instructions (LOCK Prefix, LL/SC, LDREX/STREX)

## Atomic Primitives

### Compare-and-Swap (CAS)

### Fetch-and-Add

### Test-and-Set

## Atomic Types in Languages (std::atomic, AtomicInteger)

## Memory Ordering Options (Relaxed, Acquire, Release, AcqRel, SeqCst)

## Lock-Free Programming

### Lock-Free Counters

### Atomic Reference Counting

### ABA Problem

### Hazard Pointers and Epoch-Based Reclamation

## Performance and Contention

### Cache Coherence Costs of Atomics

### Contention Scaling Issues

### False Sharing in Atomics

## See Also
- [[lock-free-programming]] — atomics are its foundation
- [[memory-ordering]] — defines visibility of atomic ops
- [[mutexes-and-locks]] — alternative to atomic synchronization
- [[race-conditions]] — atomics remedy read-modify-write races
- [[03-computer-systems/computer-architecture/cache|Cache]] — coherence drives atomic costs
