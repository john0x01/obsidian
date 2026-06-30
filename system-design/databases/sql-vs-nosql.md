# Relational (SQL) Databases vs Non-relational (NoSQL) databases

## Comparative

**Relational**
- Handles more complex relations and write-heavy applications
- Good when data changes a lot

**Non relational**
- When relations can be represented in a tree
- Handles more read-heavy applications
- Not recommended when multiple JOINs are needed
- Dynamic Schema

## Normalization and Denormalization

**Normalization**

Normalization is the process of organizing data to minimize redundancy by splitting it into separate, related collections/tables.

Pros:
- Less storage used (no duplicate data)
- Easier to update (change data in one place)
- Data consistency is maintained

Cons:
- Requires multiple queries or joins to fetch related data
- Slower reads (especially in distributed systems)

**Denormalization**

Denormalization means embedding related data together, even if it means duplicating it, to optimize for read performance.

Pros:
- Faster reads — everything in one query
- Simpler queries, no joins needed
- Better suited for distributed, high-scale systems

Cons:

- Data duplication (wastes storage)
- Harder to update (must update in multiple places)
- Risk of data inconsistency

**Why NoSQL Favors Denormalization**

NoSQL databases (MongoDB, DynamoDB, Cassandra, etc.) are designed around a key principle:

"Model your data around how you query it, not around relationships."

Unlike relational DBs, NoSQL lacks efficient joins, so denormalization is the idiomatic approach. The trade-off is intentional — you sacrifice write simplicity for massive read throughput and horizontal scalability.