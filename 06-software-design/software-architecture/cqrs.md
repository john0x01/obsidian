# CQRS (Command Query Responsibility Segregation)

## Fundamentals

### Separating Reads from Writes

### Command Model

### Query Model

## Mechanics

### Command Handlers

### Query Handlers

### Different Storage for Reads and Writes

### Materialized Views for Queries

### Eventual Consistency Between Models

### Read Model Rebuild and Replay

## Benefits

### Scaling Reads and Writes Independently

### Task-Based UIs Enabled by CQRS

## Variants and Context

### CQRS with Event Sourcing

### CQRS without Event Sourcing

### CQRS in Microservices

## Trade-offs

### Complexity Cost of CQRS

### When CQRS Is Overkill

## See Also
- [[event-sourcing]] — canonical CQRS companion
- [[event-driven-architecture]] — propagates model updates
- [[domain-driven-design]] — task-based commands from the domain
- [[05-distributed-systems/consistency-models|Consistency Models]] — eventual consistency between models
- [[06-software-design/system-design/scalability|Scalability]] — scale reads and writes separately
