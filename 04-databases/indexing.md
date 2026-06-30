# Indexing

## Fundamentals

### Why Indexes Exist
### Index Selectivity and Cardinality

## Index Structures

### B-Tree Indexes
### Hash Indexes
### Bitmap Indexes
### Inverted Indexes
### Spatial Indexes (R-Tree, Quadtree)
### GIN and GiST (PostgreSQL)

## Index Organization and Layout

### Clustered vs Non-Clustered Indexes
### Covering Indexes and Index-Only Scans
### Composite / Multi-Column Indexes

## Specialized Indexes

### Partial Indexes
### Expression / Functional Indexes
### Unique Indexes

## Maintenance and Trade-offs

### Index Maintenance Cost on Writes
### Index Bloat and Reorganization
### When Not to Index

## See Also
- [[b-trees]] — dominant index structure
- [[lsm-trees]] — write-optimized index alternative
- [[query-optimization]] — index selection during planning
- [[07-performance-engineering/database-performance|Database Performance]] — indexing for speed
- [[search-engines]] — inverted indexes in depth
