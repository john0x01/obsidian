# System Calls

## User Space vs Kernel Space

## Syscall Mechanics

### How a Syscall Works (Trap, Switch, Return)

### Syscall Tables

### Syscall Numbering and ABI

### Common Syscalls (read, write, open, close, mmap, fork, execve)

### libc Wrappers Around Syscalls

## Performance

### Syscall Overhead

### vDSO and Fast Syscalls

### io_uring as an Async Syscall Mechanism

### Batching Syscalls for Performance

## Tracing and Observability

### strace and ptrace

### eBPF for Syscall Tracing

## Security and Sandboxing

### Linux Capabilities

### seccomp and Syscall Filtering

## See Also
- [[interrupts-and-signals]] — syscalls trap via software interrupts
- [[kernel-architectures]] — the user/kernel boundary they cross
- [[io-models]] — io_uring is an async syscall mechanism
- [[instruction-set-architectures]] — the ABI builds on the ISA
- [[containers-and-namespaces]] — seccomp filters confine syscalls
