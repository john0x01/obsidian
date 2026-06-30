# CSP and Channels

## Communicating Sequential Processes (Hoare)

## Channels as First-Class Primitives

## Channel Semantics

### Buffered vs Unbuffered Channels

### Synchronous Rendezvous

### Closing Channels as Signaling

## Select / Alt Statements

## Composition Patterns

### Fan-In and Fan-Out Patterns

### Pipelines with Channels

## Flow Control and Coordination

### Backpressure via Bounded Channels

### Cancellation via Context / Done Channels

### Deadlocks with Channels

## Implementations

### Go Goroutines and Channels

### Clojure core.async

### Rust Channels (std::sync::mpsc, crossbeam)

## Comparison with Actors

## See Also
- [[actors]] — alternative message-passing model
- [[parallelism-patterns]] — fan-in/fan-out, pipelines via channels
- [[deadlock-and-livelock]] — channels can deadlock
- [[threads-and-processes]] — message passing vs shared memory
