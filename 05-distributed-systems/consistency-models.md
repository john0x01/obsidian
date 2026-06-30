# Consistency Models

## Strong Consistency Models

### Strict Consistency
### Linearizability
### Sequential Consistency
### External Consistency (Spanner)

## Transaction Isolation Models

### Serializability
### Snapshot Isolation

## Causal and Session Models

### Causal Consistency
### Causal+ Consistency
### Session Guarantees (Read Your Writes, Monotonic Reads, Monotonic Writes, Writes Follow Reads)

## Weak and Eventual Models

### Eventual Consistency
### Strong Eventual Consistency (CRDT)
### Bounded Staleness

## Cross-Cutting Concerns

### Client-Centric vs Data-Centric Models
### Per-Key vs System-Wide Consistency
### Consistency-Latency Trade-offs

## See Also
- [[cap-theorem]] — consistency under partitions
- [[pacelc]] — consistency vs latency trade-off
- [[quorum]] — R+W>N tuning for consistency
- [[crdts]] — strong eventual consistency
- [[logical-clocks]] — basis of causal consistency
- [[replication-theory]] — how replicas enforce models
