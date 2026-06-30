# Idempotency

## Fundamentals

### Idempotency Definition

### Safe vs Idempotent vs Neither

### HTTP Methods and Idempotency Contract

### Natural Idempotency (PUT vs POST)

## Idempotency Keys

### Idempotency Keys

### Storage of Idempotency Results

### Idempotency Key Expiration

### Scope (Per-User, Global, Per-Endpoint)

## Concurrency and Conflicts

### Concurrent Requests with Same Key

### Conflicting Requests with Same Key

### Retries and Exactly-Once Effects

### Deduplication Windows

## Applications

### Idempotency in Payment APIs

### Idempotency in Message Consumers

### Designing Compensations for Non-Idempotent Operations

## See Also
- [[webhooks]] — receiver idempotency for retries
- [[06-software-design/system-design/case-study-payments|Case Study: Payments]] — idempotency in payment APIs
- [[batching-and-bulk-operations]] — idempotency in batch requests
- [[rest]] — HTTP method idempotency contract
- [[04-databases/transactions-and-acid|Transactions and ACID]] — exactly-once effects
