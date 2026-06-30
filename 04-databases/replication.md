# Database Replication

## Replication Topologies

### Leader-Follower (Primary-Replica)
### Multi-Leader Replication
### Leaderless Replication

## Synchronization Modes

### Synchronous vs Asynchronous Replication
### Semi-Synchronous Replication

## Replication Methods

### Statement-Based Replication
### Row-Based Replication
### WAL/Log-Based Replication
### Trigger-Based Replication
### Logical Replication vs Physical Replication

## Consistency Guarantees

### Replication Lag
### Read-Your-Writes Consistency
### Monotonic Reads

## Failure Handling

### Failover and Promotion
### Split-Brain Scenarios
### Conflict Detection in Multi-Leader Systems

## Geographic Distribution

### Cross-Region Replication

## See Also
- [[replication-theory]] — canonical distributed-systems treatment
- [[sharding-and-partitioning]] — the complementary scaling axis
- [[write-ahead-logs]] — log shipping for replication
- [[05-distributed-systems/consistency-models|Consistency Models]] — replica consistency guarantees
- [[05-distributed-systems/quorum|Quorum]] — leaderless read/write quorums
