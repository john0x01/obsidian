# Interrupts and Signals

## Interrupts

### Hardware Interrupts

### Software Interrupts

### Maskable vs Non-Maskable Interrupts

### Interrupt Vector Table / IDT

### Interrupt Service Routines (ISRs)

### Interrupt Priorities

### Top Half vs Bottom Half

### Softirqs, Tasklets, and Workqueues

### Interrupt Coalescing

### MSI and MSI-X

## Signals

### POSIX Signals

### Common Signals (SIGINT, SIGTERM, SIGKILL, SIGSEGV, SIGCHLD)

### Real-Time Signals

### Signal Delivery and Handlers

### Signal Masks and sigprocmask

### Reentrancy and Async-Signal-Safe Functions

## See Also
- [[syscalls]] — syscalls enter the kernel via traps
- [[processes]] — signals are delivered to processes
- [[io-models]] — interrupts drive I/O completion
- [[kernel-architectures]] — interrupt handling lives in the kernel
- [[ipc]] — signals as a lightweight IPC mechanism
