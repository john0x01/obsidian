# Isolation Levels

## Fundamentals

### Why Isolation Exists

## Read Anomalies

### Dirty Reads
### Non-Repeatable Reads
### Phantom Reads
### Read Skew
### Lost Updates
### Write Skew

## Standard Isolation Levels

### Read Uncommitted
### Read Committed
### Repeatable Read
### Serializable

## Snapshot-Based Levels

### Snapshot Isolation
### Serializable Snapshot Isolation

## Applying Isolation Levels

### How Each Level Prevents Each Anomaly
### SQL Standard vs Vendor Implementations
### Choosing an Isolation Level
### Performance vs Correctness Trade-offs

## See Also
- [[transactions-and-acid]] — isolation is the I in ACID
- [[mvcc]] — implements snapshot isolation
- [[locking]] — lock-based isolation enforcement
- [[05-distributed-systems/consistency-models|Consistency Models]] — distributed analog
