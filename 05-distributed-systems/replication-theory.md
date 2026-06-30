# Replication Theory

## Fundamentals

### Purpose of Replication (Availability, Durability, Latency)
### Synchronous vs Asynchronous Replication
### Active-Active vs Active-Passive

## Replication Architectures

### State Machine Replication
### Primary-Backup Replication
### Chain Replication
### Quorum-Based Replication

## Quorum Mechanics

### Read Quorums vs Write Quorums
### Sloppy Quorums and Hinted Handoff
### Anti-Entropy and Read Repair

## Conflict Resolution

### Last-Writer-Wins
### Vector Clocks for Conflicts
### CRDT-Based Replication

## Placement and Trade-offs

### Replica Placement (Rack-Aware, Region-Aware)
### Replication Factor Trade-offs

## See Also
- [[replication]] — database replication practice
- [[quorum]] — quorum-based replication
- [[consistency-models]] — consistency replicas can enforce
- [[crdts]] — CRDT-based replication
- [[logical-clocks]] — vector clocks for conflicts
- [[consensus]] — state-machine replication
