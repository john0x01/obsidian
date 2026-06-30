# Condition Variables

## Purpose (Waiting for State Changes)

## Mechanics

### Wait, Signal, Broadcast Semantics

### Mutex Association

### Predicate Loop Pattern

## Correctness Hazards

### Spurious Wakeups

### Lost Wakeup Problem

### Thundering Herd on Broadcast

## Hoare vs Mesa Semantics Revisited

## Additional Features

### Timed Waits

### Cancellation and Interruption

## Futex-Backed Implementations

## Applications

### Combining with Queues (Bounded Buffers)

### Design Patterns (Barrier, Latch, Gate)

## Condition Variables vs Event Objects

## See Also
- [[mutexes-and-locks]] — condition variables require an associated mutex
- [[semaphores-and-monitors]] — monitors embed condition variables
- [[race-conditions]] — lost/spurious wakeups are timing hazards
- [[deadlock-and-livelock]] — missed signals can stall threads
