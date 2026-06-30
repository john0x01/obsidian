# Sockets

## Socket API Overview

## Socket Types (SOCK_STREAM, SOCK_DGRAM, SOCK_RAW)

## Address Families (AF_INET, AF_INET6, AF_UNIX)

## Socket Lifecycle (socket, bind, listen, accept, connect, close)

### Listen Backlog and SYN Queues

### Half-Close and Shutdown

## Data Transfer

### Send and Recv Semantics

### Partial Reads and Writes

### Buffer Sizes (SO_SNDBUF, SO_RCVBUF)

## Blocking vs Non-Blocking Sockets

## Socket Options (SO_REUSEADDR, SO_LINGER, TCP_NODELAY, SO_KEEPALIVE)

## Socket Multiplexing (select, poll, epoll, kqueue)

## Socket Variants

### Unix Domain Sockets

### Raw Sockets

## Error Handling (EAGAIN, EWOULDBLOCK, EPIPE)

## See Also
- [[tcp]] — SOCK_STREAM exposes TCP
- [[udp]] — SOCK_DGRAM exposes UDP
- [[ipc]] — Unix domain sockets are IPC
- [[03-computer-systems/operating-systems/io-models|I/O Models]] — blocking vs non-blocking, epoll
- [[03-computer-systems/operating-systems/syscalls|Syscalls]] — socket calls are syscalls
