# Inter-Process Communication (IPC)

## Why Processes Need to Communicate

## Channel-Based IPC

### Pipes (Anonymous, Named/FIFO)

### Unix Domain Sockets

### Network Sockets for IPC

### Message Queues

## Shared-Memory IPC

### Shared Memory (POSIX, System V)

### Memory-Mapped Files as IPC

## Synchronization and Notification

### Semaphores for Synchronization

### Signals as IPC

### eventfd and signalfd

## Higher-Level Mechanisms

### D-Bus

### Binder (Android IPC)

### RPC Mechanisms

## Trade-offs

### Copy Semantics vs Shared Memory Trade-offs

### Ordering and Delivery Guarantees

## See Also
- [[sockets]] — Unix and network sockets for IPC
- [[message-queues]] — queue-based message passing
- [[processes]] — IPC connects separate processes
- [[interrupts-and-signals]] — signals as a lightweight IPC
- [[03-computer-systems/operating-systems/memory-management|Memory Management (OS)]] — shared memory and mmap
