# Case Study: Rate Limiter

## Requirements

### Requirements (Global, Per-User, Per-Endpoint)

### Scale Estimation

## Algorithm Choice

### Algorithm Choice (Token Bucket, Leaky Bucket, Sliding Window)

### Burst Allowance

### Soft vs Hard Limits

## State Management

### In-Memory vs Distributed State

### Redis-Based Implementation

### Atomicity with Lua Scripts

### Clock Skew Handling

## Enforcement

### Client-Side vs Gateway-Side Enforcement

### Hierarchical / Composite Limits

### Bypass and Allow-Listing

## Trade-offs

### Accuracy vs Performance Trade-offs

### Handling Redis Failure

## Client Contract

### Rate Limit Headers Design

### Response Codes and Retry-After

## Observability

### Metrics and Observability
