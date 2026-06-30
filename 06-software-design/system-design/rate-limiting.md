# Rate Limiting (System Design)

## Fundamentals

### Purpose at System Scale

### Edge vs Service-Level Rate Limiting

## Algorithms

### Token Bucket Algorithm

### Leaky Bucket Algorithm

### Fixed Window Counter

### Sliding Window Log

### Sliding Window Counter

## Distributed Enforcement

### Distributed Rate Limiting with Redis

### Lua Scripts for Atomicity

### Hierarchical Limits (Global, User, Endpoint)

## Variants

### Adaptive Rate Limiting

### Quota vs Rate Limits

### Concurrency Limits (In-Flight Caps)

## Related Techniques

### Load Shedding as Related Technique

### Backpressure-Based Limiting

### Gray Failures and Rate Limiting

## Client Communication

### Rate Limiting Headers to Clients

## See Also
- [[06-software-design/api-design/rate-limiting|Rate Limiting (API)]] — API-level rate limiting
- [[case-study-rate-limiter]] — worked rate-limiter design
- [[high-availability]] — load shedding and backpressure
- [[load-balancing]] — enforcement at the edge
