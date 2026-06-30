# Event Sourcing

## Fundamentals

### State as a Stream of Events

### Event Store Requirements

### Appending Events

### Rebuilding State via Replay

## Read Side

### Projections / Read Models

### Eventual Consistency of Projections

### Temporal Queries and Time Travel

### Audit Log as a Free Feature

## Performance and Concurrency

### Snapshots for Performance

### Concurrency Control (Optimistic, Stream Version)

## Schema Evolution

### Event Versioning and Upcasters

### Event Schema Evolution

## Correcting Data

### Handling Bugs via Event Correction

### Compensating Events

## Ecosystem and Fit

### Event Sourcing with CQRS

### Tooling (EventStoreDB, Axon, Marten)

### When Event Sourcing Is the Wrong Choice

## See Also
- [[cqrs]] — frequently paired read/write split
- [[event-driven-architecture]] — broader event paradigm
- [[domain-driven-design]] — domain events drive the log
- [[06-software-design/system-design/message-queues|Message Queues]] — event transport
- [[04-databases/transactions-and-acid|Transactions and ACID]] — append-only vs mutable state
