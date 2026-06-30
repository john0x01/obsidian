# Distributed Transactions

## Fundamentals

### Why Distributed Transactions Are Hard
### ACID Across Nodes
### The Dual-Write Problem
### Idempotency Requirements

## Atomic Commit Protocols

### Two-Phase Commit (2PC)
### Three-Phase Commit (3PC)
### Timeouts and In-Doubt States
### XA and JTA

## Application-Level Patterns

### Saga Pattern (Choreography vs Orchestration)
### Compensation Actions
### TCC (Try-Confirm-Cancel)
### Transactional Outbox and Event Publishing

## Modern and Research Systems

### Percolator Transactions
### Google Spanner Transactions
### Calvin Deterministic Transactions

## See Also
- [[two-phase-commit]] — core atomic commit protocol
- [[transactions-and-acid]] — single-node ACID foundation
- [[consistency-models]] — serializability and isolation
- [[consensus]] — consensus-backed commit
