# Mutexes and Locks

## Critical Sections

## Mutex Semantics

## Lock Implementations

### Spinlocks

### Adaptive Mutexes (Spin-then-Block)

### Futex-Based Mutexes (Linux)

## Mutex Variants

### Recursive (Reentrant) Mutexes

### Reader-Writer Locks

### Upgradable Locks

### Fair vs Unfair Locks

## Try-Lock and Timed-Lock

## Lock Contention

## Lock Granularity (Coarse vs Fine)

## Lock Hierarchies to Prevent Deadlock

## Priority Inversion and Inheritance

## Lock-Free Alternatives

## See Also
- [[semaphores-and-monitors]] — higher-level mutual exclusion
- [[condition-variables]] — pair with mutexes to wait on state
- [[deadlock-and-livelock]] — lock misuse causes deadlock
- [[race-conditions]] — locks guard critical sections
- [[lock-free-programming]] — the lockless alternative
