# Message Ordering

## Fundamentals

### Why Ordering Is Hard in Distributed Systems
### Happens-Before vs Wall Clock
### Reordering at Network and Application Layers

## Ordering Guarantees

### FIFO Ordering
### Causal Ordering
### Total Ordering
### Causal-Total Ordering

## Mechanisms

### Sequence Numbers per Publisher
### Vector Clocks for Causal Ordering
### Sequencers for Total Order
### Atomic Broadcast
### Timestamp-Based Ordering Pitfalls

## Delivery Semantics and Edge Cases

### Kafka's Partition-Level Ordering
### Out-of-Order Handling Strategies
### Exactly-Once Delivery Requires Ordering + Dedup

## See Also
- [[logical-clocks]] — vector clocks for causal ordering (canonical)
- [[consistency-models]] — causal and total order guarantees
- [[consensus]] — atomic broadcast and total order
