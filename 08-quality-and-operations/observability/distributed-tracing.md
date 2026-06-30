# Distributed Tracing

## Fundamentals

### Trace, Span, and Span Context

### Parent-Child Span Relationships

## Mechanics

### Trace Propagation Headers (W3C traceparent, B3)

### Trace Enrichment (Attributes, Events, Links)

### Async and Queue Span Linking

## Sampling

### Sampling Strategies (Head-Based, Tail-Based, Probabilistic, Adaptive)

## Analysis

### Service Map Generation

### Latency Breakdown Analysis

### Critical Path Analysis

### Root Cause Analysis with Traces

## Correlation

### Trace-Log Correlation

### Trace-Metric Correlation (Exemplars)

## Instrumentation

### Manual vs Automatic Instrumentation

### Performance Overhead of Instrumentation

## Tooling and Operations

### OpenTelemetry Tracing

### Jaeger and Zipkin

### Storage and Retention of Traces

## See Also
- [[opentelemetry]] — canonical instrumentation standard
- [[correlation-ids]] — request identity underpinning spans
- [[metrics]] — trace-metric correlation via exemplars
- [[logging]] — trace-log correlation
- [[microservices]] — why cross-service tracing matters
