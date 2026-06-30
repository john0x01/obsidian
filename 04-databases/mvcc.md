# MVCC (Multi-Version Concurrency Control)

## Fundamentals

### Motivation (Readers Don't Block Writers)
### MVCC vs Locking-Based Concurrency

## Core Mechanics

### Version Chains
### Transaction IDs / Timestamps
### Snapshot Creation
### Visibility Rules

## Consistency Behavior

### Read-Your-Own-Writes
### Write-Write Conflicts Under MVCC

## Version Cleanup

### Vacuum / Garbage Collection (PostgreSQL)
### Undo Segments (Oracle, MySQL InnoDB)
### Bloat from Dead Tuples
### Long Transactions and Version Retention

## Implementations

### MVCC Implementation in PostgreSQL
### MVCC Implementation in InnoDB
### HOT Updates

## MVCC and Replication

## See Also
- [[isolation-levels]] — MVCC implements snapshot isolation
- [[locking]] — the alternative MVCC avoids
- [[transactions-and-acid]] — concurrency control context
- [[nosql-document]] — CouchDB MVCC implementation
