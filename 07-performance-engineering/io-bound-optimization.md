# I/O-Bound Optimization

## Identifying I/O-Bound Workloads

## Async I/O (epoll, kqueue, io_uring)

## Buffering and Batching

## Prefetching

### Pre-Fetching

### Read-Ahead and Write-Behind

## Disk and Filesystem

### Page Cache Utilization

### Direct I/O When Appropriate

### mmap for Random Access

### Zero-Copy (sendfile, splice)

### Disk I/O Patterns (Sequential vs Random)

## Network

### Pipelining Requests

### Connection Pooling

### Keep-Alive for HTTP

### Multiplexing (HTTP/2, gRPC)

### Network Tuning (TCP Window, MTU)

### DNS Caching

## Compression Trade-offs

## See Also
- [[cpu-bound-optimization]] — sibling bottleneck class
- [[memory-bound-optimization]] — sibling bottleneck class
- [[io-models]] — async I/O models (epoll, kqueue, io_uring)
- [[network-performance]] — network-side I/O tuning
- [[batching]] — buffering and batching I/O
