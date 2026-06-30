# Memory-Bound Optimization

## Memory Wall and Bandwidth Limits

## Working Set Size Awareness

## Locality Optimization

### Data Locality Optimization

### Cache-Aware Algorithms

### Cache-Oblivious Algorithms

## Data Layout

### Struct Packing and Alignment

### Array of Structs vs Struct of Arrays

## Reducing Allocation and Copies

### Pre-Allocation and Object Pooling

### Reducing Allocation Churn (GC Pressure)

### Avoiding Unnecessary Copies

### Move Semantics

### Zero-Copy Techniques

## Allocator Strategies

### Arena / Bump Allocators

### Custom Allocators

## OS and Hardware Tuning

### Huge Pages

### NUMA Locality

### Memory Compression

## See Also
- [[cpu-bound-optimization]] — sibling bottleneck class
- [[io-bound-optimization]] — sibling bottleneck class
- [[data-locality]] — canonical note on locality optimization
- [[cache-friendly-code]] — cache-aware layout and access
- [[memory-hierarchy]] — bandwidth and the memory wall
- [[garbage-collection]] — reducing GC allocation pressure
