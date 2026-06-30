# Sharding Strategies

## Motivation

### Why Shard (Write Scalability, Data Size)

## Partitioning Schemes

### Range-Based Sharding

### Hash-Based Sharding

### Directory-Based Sharding

### Geographic Sharding

### Tenant-Based Sharding (Multi-Tenant)

### Consistent Hashing for Minimal Rebalancing

## Shard Key Design

### Choosing a Shard Key

### Hot Shards and Skew

### Salting Keys to Spread Load

## Querying Across Shards

### Cross-Shard Queries and Joins

### Scatter-Gather Patterns

### Cross-Shard Transactions

### Global Secondary Indexes Across Shards

## Operations

### Resharding Live Systems

### Dual-Writing During Migration

### Observability per Shard

## See Also
- [[sharding-and-partitioning]] — canonical DB sharding note
- [[partitioning]] — distributed partitioning
- [[scalability]] — sharding for write scale
- [[replication]] — replication alongside sharding
