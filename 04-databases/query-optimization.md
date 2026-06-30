# Query Optimization

## Query Processing Pipeline

### Query Parser and Rewriter
### Logical Plan vs Physical Plan

## Optimizer Types

### Rule-Based Optimizer
### Cost-Based Optimizer

## Cost Estimation

### Statistics and Histograms
### Cardinality Estimation

## Plan Transformations

### Join Algorithms (Nested Loop, Hash Join, Merge Join)
### Join Ordering
### Predicate Pushdown
### Projection Pushdown
### Subquery Unnesting
### Index Selection

## Execution and Caching

### Parallel Query Execution
### Caching of Plans (Plan Cache)

## Tooling and Tuning

### EXPLAIN and EXPLAIN ANALYZE
### Query Hints
### Common Performance Anti-Patterns

## See Also
- [[indexing]] — index selection drives plans
- [[07-performance-engineering/database-performance|Database Performance]] — tuning slow queries
- [[b-trees]] — index access methods
- [[sql-fundamentals]] — the queries being optimized
- [[oltp-vs-olap]] — workload shapes optimization
