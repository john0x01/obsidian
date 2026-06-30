# Transactions and ACID

## ACID Properties

### Atomicity
### Consistency
### Isolation
### Durability

## Transaction Control

### Begin, Commit, Rollback
### Savepoints
### Implicit vs Explicit Transactions
### Nested Transactions

## Distributed Transactions

### Distributed Transactions
### Two-Phase Commit

## Logging and Recovery

### Logging and Recovery (ARIES)
### Undo and Redo Logs
### Write-Ahead Logging
### Checkpoints
### Crash Recovery

## Pitfalls and Edge Cases

### Long-Running Transactions Pitfalls
### Autonomous Transactions

## See Also
- [[isolation-levels]] — the I in ACID, in depth
- [[locking]] — enforcing isolation via locks
- [[mvcc]] — lock-free isolation mechanism
- [[write-ahead-logs]] — durability and crash recovery
- [[two-phase-commit]] — atomicity across nodes
- [[distributed-transactions]] — ACID across services
