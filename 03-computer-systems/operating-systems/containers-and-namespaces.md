# Containers and Namespaces

## What a Container Really Is

## Kernel Isolation Primitives

### Linux Namespaces (PID, Network, Mount, UTS, IPC, User, Cgroup, Time)

### Control Groups (cgroups v1 vs v2)

### Capabilities

### chroot and Its Limits

## Container Filesystems

### OverlayFS and Union Filesystems

### Container Images and Layers

## Standards and Runtimes

### OCI Runtime Spec

### runc, containerd, and CRI-O

### Docker vs Podman Architectures

### Rootless Containers

## Security Isolation (seccomp, AppArmor, SELinux)

## gVisor and Kata Containers

## Container Networking (bridge, host, overlay)

## Volume Mounts and Persistence

## See Also
- [[containers]] — the DevOps/runtime view of containers
- [[processes]] — namespaces isolate process trees
- [[kernel-architectures]] — isolation primitives live in the kernel
- [[syscalls]] — seccomp filters the syscall surface
- [[03-computer-systems/operating-systems/memory-management|Memory Management (OS)]] — cgroups cap container memory
