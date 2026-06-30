# Write-Ahead Logs (WAL)

## Fundamentals

### WAL Principle (Log Before Data Page)
### Durability Guarantee

## Log Structure

### Redo vs Undo Logging
### Physical vs Logical Logging
### Log Sequence Numbers (LSN)

## Write and Flush Mechanics

### Flush Policy (fsync, group commit)
### Log Buffer and Background Writers
### Checkpoints and Fuzzy Checkpoints

## Recovery

### ARIES Algorithm
### Point-in-Time Recovery
### WAL Corruption and Recovery

## Replication and Shipping

### WAL-Based Replication
### WAL Shipping

## Storage Management

### WAL Compression
### Log Archiving and Retention

## Comparison

### Comparison with Command Logging

## See Also
- [[transactions-and-acid]] — WAL provides durability
- [[replication]] — WAL/log-based replication
- [[lsm-trees]] — WAL precedes MemTable flush
- [[mvcc]] — versioning and recovery interplay
