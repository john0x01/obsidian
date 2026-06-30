# Webhooks

## Fundamentals

### Event-Driven API Paradigm

### Subscription Management

### Event Types and Payloads

## Security

### HMAC Signatures for Verification

### Replay Attack Prevention (Timestamps, Nonces)

### Webhook Security (IP Allow Lists, mTLS)

## Delivery Semantics

### Delivery Guarantees (At-Least-Once)

### Retry Policies and Exponential Backoff

### Receiver Idempotency

### Ordering Guarantees (or Lack Thereof)

### Delivery Failures and Dead Letter Queues

## Operations

### Observability (Delivery Logs, Dashboards)

### Rate Limits on Inbound Webhooks

### Versioning Webhook Payloads

### Testing Webhooks (Tunnels, Mock Endpoints)

## Scaling

### Fan-Out Architecture for Publishers

## See Also
- [[idempotency]] — receiver idempotency for at-least-once delivery
- [[event-driven-architecture]] — webhooks as HTTP event delivery
- [[06-software-design/system-design/pub-sub|Pub/Sub]] — fan-out publishing model
- [[06-software-design/system-design/case-study-payments|Case Study: Payments]] — payment event notifications
- [[08-quality-and-operations/security/csrf|CSRF]] — HMAC signing and replay defense
