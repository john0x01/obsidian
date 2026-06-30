# Rate Limiting

## Motivation

### Purpose (Abuse Prevention, Fair Use, Cost Control)

## Algorithms

### Fixed Window

### Sliding Window Log

### Sliding Window Counter

### Token Bucket

### Leaky Bucket

### Concurrent Request Limits

## Scoping Limits

### Global vs Per-User vs Per-IP Limits

### Per-Endpoint Limits

### Tiered Limits by Plan

## Client Signaling

### Headers for Rate Limit State (X-RateLimit-*, RateLimit Standard)

### 429 Too Many Requests Response

### Retry-After Header

## Advanced Topics

### Distributed Rate Limiting (Redis, Token Bucket at Edge)

### Burst Allowance

### Soft vs Hard Limits

### Bypass for Trusted Clients

## See Also
- [[06-software-design/system-design/rate-limiting|Rate Limiting (system design)]] — canonical system-design treatment
- [[06-software-design/system-design/case-study-rate-limiter|Case Study: Rate Limiter]] — worked design example
- [[api-gateways]] — common enforcement point
- [[error-handling]] — 429 and Retry-After responses
- [[08-quality-and-operations/security/ddos-mitigation|DDoS Mitigation]] — abuse-prevention overlap
