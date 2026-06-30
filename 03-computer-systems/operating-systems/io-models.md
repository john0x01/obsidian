# I/O Models

## Synchronous I/O Models

### Blocking I/O

### Non-Blocking I/O

### I/O Multiplexing (select, poll)

### Event-Driven I/O (epoll, kqueue)

### Signal-Driven I/O

## Asynchronous I/O (POSIX AIO, io_uring)

## Design Patterns

### Readiness vs Completion Models

### Reactor vs Proactor Patterns

## Data Path

### Direct I/O vs Buffered I/O

### Zero-Copy (sendfile, splice, MSG_ZEROCOPY)

### DMA (Direct Memory Access)

## Scheduling and Performance

### I/O Schedulers (CFQ, Deadline, BFQ, mq-deadline)

### Write Amplification

### Batching and Pipelining I/O

### Kernel Bypass (DPDK, SPDK)

## See Also
- [[io-bound-optimization]] — practical tuning of I/O-bound code
- [[event-loops]] — reactor pattern built on epoll/kqueue
- [[syscalls]] — io_uring and read/write are syscalls
- [[scheduling]] — I/O-bound vs CPU-bound workloads
- [[async-and-await]] — async I/O surfaced to application code
