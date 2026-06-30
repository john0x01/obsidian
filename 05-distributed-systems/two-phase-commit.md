# Two-Phase Commit (2PC)

## Protocol Mechanics

### Coordinator and Participants
### Prepare Phase
### Commit Phase
### 2PC Logging Requirements

## Failure Handling

### Blocking Nature of 2PC
### Coordinator Failure Scenarios
### Participant Failure Scenarios
### In-Doubt Transactions
### Presumed Abort and Presumed Commit

## Limitations

### 2PC in Microservices Criticism
### Limits on Scale and Availability

## Alternatives

### Three-Phase Commit (3PC)
### Sagas
### Percolator Transactions
### XA Transactions and Distributed Transactions

## See Also
- [[distributed-transactions]] — canonical parent; sagas and alternatives
- [[transactions-and-acid]] — ACID guarantees 2PC enforces
- [[consensus]] — atomic commit vs consensus
