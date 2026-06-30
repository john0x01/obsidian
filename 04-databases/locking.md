# Database Locking

## Lock Fundamentals

### Lock Modes (Shared, Exclusive, Update, Intent)
### Lock Granularity (Row, Page, Table, Database)
### Intention Locks
### Lock Escalation

## Locking Protocols

### Two-Phase Locking (2PL)
### Strict 2PL
### Optimistic vs Pessimistic Locking

## Range and Predicate Locks

### Predicate Locks
### Range Locks and Next-Key Locks
### Gap Locks (InnoDB)

## Deadlocks

### Deadlocks in the Database
### Deadlock Detection and Victim Selection
### Lock Timeouts

## Application-Level Locking

### Application-Level Advisory Locks
### SELECT FOR UPDATE / SKIP LOCKED

## Performance

### Lock Contention Hotspots

## See Also
- [[isolation-levels]] — locks enforce isolation
- [[mvcc]] — lock-free concurrency alternative
- [[transactions-and-acid]] — concurrency control context
- [[03-computer-systems/concurrency-and-parallelism/deadlock-and-livelock|Deadlock and Livelock]] — general deadlock theory
- [[03-computer-systems/concurrency-and-parallelism/mutexes-and-locks|Mutexes and Locks]] — in-memory locking
