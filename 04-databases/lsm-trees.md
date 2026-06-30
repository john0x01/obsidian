# LSM Trees

## Fundamentals

### Log-Structured Merge Concept

## Read and Write Paths

### Write Path (MemTable, Flush, SSTable)
### Read Path (MemTable + SSTables + Bloom Filters)
### Bloom Filters for Negative Lookups

## Compaction

### Compaction Overview
### Leveled Compaction
### Tiered (Size-Tiered) Compaction
### Hybrid Strategies

## Amplification Trade-offs

### Write Amplification
### Read Amplification
### Space Amplification

## Deletes and Expiry

### Tombstones and Deletes
### TTL Support

## Optimization and Comparison

### Optimization with Key Prefixes
### LSM vs B-Tree Trade-offs

## Implementations

### LSM in RocksDB and LevelDB
