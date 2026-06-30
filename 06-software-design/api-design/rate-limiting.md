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
