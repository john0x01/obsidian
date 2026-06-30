# SQL vs NoSQL

## Comparative Overview

**Relational (SQL) databases** organize data into tables with fixed schemas and enforce relationships between them.

- Handle complex relationships and write-heavy workloads well.
- Suited to data that changes frequently and must stay consistent.

**Non-relational (NoSQL) databases** store data in flexible structures (documents, key-value pairs, columns, or graphs) rather than fixed tables.

- Best when relationships can be represented as a tree (hierarchical, self-contained data).
- Suited to read-heavy workloads.
- Not recommended when queries require multiple JOINs.
- Use a dynamic (flexible) schema.

## Normalization and Denormalization

**Normalization** is the process of organizing data to minimize redundancy by splitting it into separate, related tables or collections.

Pros:
- Less storage used (no duplicate data).
- Easier to update (data lives in one place).
- Data consistency is maintained.

Cons:
- Requires multiple queries or joins to fetch related data.
- Slower reads, especially in distributed systems.

**Denormalization** means embedding related data together — even if it duplicates data — to optimize read performance.

Pros:
- Faster reads, since everything is available in one query.
- Simpler queries with no joins needed.
- Better suited to distributed, high-scale systems.

Cons:
- Data duplication wastes storage.
- Harder to update, since the same data must be changed in multiple places.
- Risk of data inconsistency.

## Why NoSQL Favors Denormalization

NoSQL databases (MongoDB, DynamoDB, Cassandra, and others) are designed around a single principle:

> Model your data around how you query it, not around relationships.

Unlike relational databases, NoSQL systems lack efficient joins, so denormalization is the idiomatic approach. The trade-off is intentional: you sacrifice write simplicity in exchange for high read throughput and horizontal scalability.

## See Also
- [[relational-model]] — the SQL data model
- [[normalization]] — normalize vs denormalize trade-off
- [[nosql-document]] — a major NoSQL family
- [[nosql-key-value]] — a major NoSQL family
- [[nosql-graph]] — a major NoSQL family
- [[nosql-columnar]] — a major NoSQL family
