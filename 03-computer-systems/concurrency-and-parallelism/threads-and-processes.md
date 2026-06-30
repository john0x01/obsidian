# Threads and Processes

## Fundamentals

### Processes as Isolation Boundaries

### Threads as Execution Units Within a Process

### Concurrency vs Parallelism

## Communication and Memory

### Shared Memory vs Message Passing

## Costs and Trade-Offs

### Cost of Creation (Threads Cheaper, Processes Heavier)

### Context Switch Overhead Considerations

### When to Use Threads vs Processes

## Architectural Models

### Multi-Process Architectures (Pre-fork, Worker Pools)

### Multi-Threaded Architectures

### Hybrid Architectures

### Thread-Per-Request vs Event Loops

## Advanced Topics

### Green Threads / Fibers / Coroutines

### GIL and Its Implications (Python, Ruby)

### Per-Core Pinning Strategies

## See Also
- [[threads]] — OS thread internals (canonical)
- [[processes]] — OS process internals (canonical)
- [[03-computer-systems/operating-systems/scheduling|Scheduling]] — how the OS dispatches them
- [[03-computer-systems/operating-systems/ipc|IPC]] — inter-process communication
- [[thread-pools]] — managing reusable threads
