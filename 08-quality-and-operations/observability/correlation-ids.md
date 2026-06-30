# Correlation IDs

## Fundamentals

### Purpose of Correlation

### Request ID vs Trace ID vs Correlation ID Distinctions

### Session IDs vs Correlation IDs

### User-Visible vs Internal IDs

## Generation

### Generation Strategies (UUID, Snowflake, ULID)

## Propagation

### Propagation Across Services

### Propagation in HTTP Headers

### Propagation in Message Queues

### Propagation in Async Tasks

### Correlation in Webhooks

### Correlation in Batch Jobs

## Relationships and Causation

### Parent / Child Relationships

### Causation IDs in Event Sourcing

## Usage

### Correlation with Logs, Metrics, Traces

### Debugging Workflows Using Correlation IDs

## Operational Concerns

### Privacy Considerations

## See Also
- [[distributed-tracing]] — trace/span context vs correlation IDs
- [[opentelemetry]] — context propagation and baggage
- [[logging]] — contextual log fields
- [[debugging-production]] — root-causing incidents by ID
- [[event-sourcing]] — causation IDs in event flows
