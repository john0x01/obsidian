# Columnar Databases

## Fundamentals

### Row-Oriented vs Column-Oriented Storage
### Compression Benefits of Columnar

## Wide-Column Stores

### Wide-Column Stores (Cassandra, HBase, Bigtable)
### Column Families
### Partition and Clustering Keys
### SSTables and LSM in Columnar
### Tombstones in Wide-Column Stores
### Read and Write Paths in Cassandra
### CQL and SQL-Like Interfaces

## Columnar Analytical Stores

### Columnar Analytical Stores (ClickHouse, Vertica, Parquet)
### Data Encoding (Dictionary, RLE, Delta)
### Vectorized Execution
### Projection Pruning
### Late Materialization

## Data Modeling and Workloads

### Denormalized Data Modeling
### Time-Series Workload Fit

## See Also
- [[oltp-vs-olap]] — column storage powers OLAP
- [[data-warehousing]] — columnar analytical stores
- [[lsm-trees]] — write path of wide-column stores
- [[time-series-databases]] — columnar time-series fit
- [[nosql-key-value]] — sibling NoSQL family
