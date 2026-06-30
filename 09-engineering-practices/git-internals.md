# Git Internals

## Storage Model

### Content-Addressable Storage
### SHA-1 and SHA-256 Hashing
### Objects (Blob, Tree, Commit, Tag)
### Packfiles and Delta Compression

## Repository Layout

### The .git Directory Layout
### Working Tree vs Index vs Repository
### Index / Staging Area
### Refs and Refspecs
### HEAD and Detached HEAD
### Reflog Internals

## Commands

### Plumbing vs Porcelain Commands
### Git Hooks

## Integration Operations

### Merge Base and Three-Way Merge
### Rebase Internals
### Garbage Collection (git gc)

## Remote Access

### Remote Protocols (HTTP, SSH, Git Protocol)
### Shallow Clones
### Sparse Checkout and Partial Clone

## See Also
- [[version-control]] — canonical user-facing Git workflows
- [[branching-strategies]] — branches as refs over the object model
- [[hashing]] — SHA-1/SHA-256 content addressing
- [[b-trees]] — packfile and index data structures
