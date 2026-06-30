# Memory Models

## What a Memory Model Defines

## Sources of Reordering

### Compiler Reordering

### CPU Reordering

### Load-Load, Load-Store, Store-Store, Store-Load Orderings

## Consistency Models

### Sequential Consistency

### Total Store Ordering (TSO) — x86 Model

### Weak Memory Models (ARM, POWER)

### Release Consistency

## Ordering Primitives

### Memory Barriers / Fences

### Atomic Operations and Memory Ordering

### Acquire-Release Semantics

### Happens-Before Relationship

## Data Races vs Benign Races

## Programming Language Memory Models (C++, Java, Go)

## Double-Checked Locking Pitfalls

## See Also
- [[memory-ordering]] — the concurrency view of ordering (canonical)
- [[atomic-operations]] — atomics carry ordering semantics
- [[mutexes-and-locks]] — locks establish happens-before
- [[lock-free-programming]] — relies on fences and acquire-release
- [[cpu]] — CPU reordering is a source of weak ordering
