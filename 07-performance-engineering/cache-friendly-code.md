# Cache-Friendly Code

## Understanding Cache Hierarchy Impact

## Sequential Access Patterns

## Cache Line Alignment

## False Sharing Between Cores

### Padding to Avoid False Sharing

## Loop and Blocking Transformations

### Cache Blocking / Tiling

### Loop Interchange for Locality

### Cache-Oblivious Patterns

## Hot/Cold Data Separation

### Hot Data in Frequently Accessed Fields

### Cold Data Separation

## Avoiding Pointer Indirection

### Inline Small Types

## Data Prefetching

## Branch Prediction Friendliness

## Working Set Minimization

## TLB Misses and Huge Pages

## Profiling Cache Misses (perf, VTune)

## See Also
- [[data-locality]] — canonical note on locality principles
- [[memory-bound-optimization]] — broader memory-bottleneck context
- [[cache]] — hardware cache fundamentals
- [[memory-hierarchy]] — why cache levels matter
- [[branch-prediction]] — branch-prediction-friendly code
