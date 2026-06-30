# Raft

## Fundamentals

### Design Goal (Understandability)
### Strong Leader Concept
### Term Numbers

## Leader Election

### Election Process
### Election Timeouts and Randomization
### Pre-Vote Optimization

## Log Replication

### AppendEntries RPC
### Log Matching Property
### Commit Rules
### State Machine Application

## Advanced Mechanics

### Log Compaction and Snapshots
### Membership Changes (Joint Consensus, Single-Server Changes)
### Leader Leases
### Read-Only Queries (Linearizable Reads)

## Practice

### Raft Implementations (etcd, Consul, TiKV)
### Failure Scenarios and Recovery
