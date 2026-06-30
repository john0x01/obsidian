# Event-Driven Architecture

## Fundamentals

### Events as First-Class Citizens

### Event Producers and Consumers

### Pub/Sub Semantics

## Infrastructure

### Event Brokers (Kafka, Pulsar, NATS, RabbitMQ)

### Event Streaming vs Messaging

## Coordination

### Choreography vs Orchestration

## Contracts and Evolution

### Event Schemas and Contracts

### Event Versioning and Evolution

## Delivery Semantics

### Delivery Guarantees (At-Most-Once, At-Least-Once, Exactly-Once)

### Ordering Guarantees

### Idempotent Consumers

### Dead Letter Queues

### Backpressure Handling

## Consistency and Reliability Patterns

### Eventual Consistency Implications

### Transactional Outbox Pattern

### Event Sourcing Overlap

## Architectural Context

### Reactive Systems Manifesto Alignment

### Event-Driven Anti-Patterns

## See Also
- [[event-sourcing]] — persists state as an event log
- [[cqrs]] — pairs commands with event-driven reads
- [[06-software-design/system-design/pub-sub|Pub/Sub]] — core delivery semantics
- [[06-software-design/system-design/message-queues|Message Queues]] — broker infrastructure
- [[06-software-design/api-design/webhooks|Webhooks]] — event delivery over HTTP
- [[reactive-programming]] — reactive systems alignment
