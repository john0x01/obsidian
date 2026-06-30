# Time-Series Databases

## Fundamentals

### Time-Series Data Characteristics
### High Ingestion Rates
### Out-of-Order Writes

## Data Modeling

### Tags vs Fields (Metric Metadata)
### Cardinality Explosion

## Storage and Retention

### Time-Based Partitioning
### Compression Techniques (Delta, Gorilla, XOR)
### Retention Policies
### Downsampling and Rollups

## Querying

### Continuous Queries
### Window Functions on Time Series
### PromQL, Flux, and SQL-Based TSDBs

## Implementations

### Prometheus Architecture
### InfluxDB Architecture
### TimescaleDB (PostgreSQL Extension)
### VictoriaMetrics, QuestDB

## Use Cases

### Use Cases (Metrics, IoT, Finance)

## See Also
- [[nosql-columnar]] — columnar storage fits time-series
- [[data-warehousing]] — adjacent analytical store
- [[sharding-and-partitioning]] — time-based partitioning
- [[08-quality-and-operations/observability/metrics|Metrics]] — primary TSDB consumer
