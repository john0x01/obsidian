# Hash Tables

A hash table stores key–value associations in a flat array of *buckets*, using a hash function to compute, from the key alone, the index where its entry lives. It solves the problem of near-constant-time membership, lookup, insertion, and deletion without keeping keys in sorted order — trading the ordering (and worst-case guarantees) of a balanced tree for raw average-case speed.

## The Idea: Key → Hash → Bucket

The mechanism is a two-stage map. First a [[hash-functions|hash function]] turns an arbitrary key into a wide integer; then that integer is folded into the array's index space, classically by modulo:

```ts
index = hash(key) % capacity;   // capacity = number of buckets
```

The array slot at `index` is a *bucket*. Insertion computes the index and drops the entry there; lookup recomputes the same index and reads it back. Because the index is derived purely arithmetically from the key, we jump straight to the data — no scanning, no comparisons across the whole set. That is the entire source of the O(1) claim.

Two distinct keys can hash to the same bucket — a *collision* — which every real table must handle (see [[collision-resolution]]). The choice of scheme is what makes the difference between a fast table and a pathological one.

## Complexity and the Assumption Behind It

- **Search / insert / delete: O(1) average, O(n) worst.**

The average bound is not free — it rests on the **simple uniform hashing assumption (SUHA)**: each key is equally likely to land in any bucket, independently of the others. Under SUHA, the expected number of keys per bucket is the **load factor** α = n / m (n entries, m buckets), so an average lookup touches O(1 + α) entries. Keep α bounded by a constant and every operation is expected O(1).

The **worst case is O(n)**: if the hash function (or an adversary) funnels every key into one bucket, the table degenerates into a linear scan of a single chain. SUHA is an *assumption about the input distribution and the hash*, not a theorem — which is exactly the crack that hash-flooding attacks exploit.

Space is O(n + m). You keep m proportional to n to hold α down, so total space stays O(n).

## Load Factor and Dynamic Resizing

The load factor α is the tuning dial. Let it grow and average chain length (or probe distance) grows with it, degrading every operation. So tables **resize**: when α crosses a threshold (typically 0.75 for chaining, lower ~0.5 for open addressing), allocate a larger array — usually **doubling** m — and **rehash** every existing entry into the new array, because `hash(key) % capacity` changes when `capacity` does.

A single rehash is O(n), which sounds alarming, but it happens rarely. Because capacity doubles, the O(n) cost is spread across the Θ(n) cheap insertions that preceded it, giving **amortized O(1)** per insertion — the same geometric-growth argument that powers [[dynamic-arrays]]. See [[amortized-analysis]] for the aggregate/potential proofs. The senior caveat: individual inserts still spike to O(n), so amortized-O(1) is not the same as worst-case-O(1) — a latency-sensitive path may see a tail spike right when the table doubles.

## No Ordering

A hash table imposes **no order** on its keys — neither insertion order nor sort order. Iteration visits buckets in array layout, which is effectively arbitrary and can shuffle on resize. If you need ordered iteration, range queries, or predecessor/successor, you want a balanced tree ([[binary-search-trees]], [[red-black-trees]]) or an insertion-ordered wrapper — not a bare hash table.

## Senior Pitfalls

- **The O(n) worst case is real, and weaponizable.** With a weak, publicly-known hash, an attacker crafts many keys that collide into one bucket, turning every insert into O(n) and the whole table into O(n²) to fill — a **hash-flooding denial-of-service**. This famously took down web frameworks whose form parameters landed in a predictable hash map. Defenses: a **keyed, randomized hash** (SipHash) seeded per-process so an attacker can't predict buckets, and switching a hot bucket from a list to a tree (Java 8+ treeifies long chains to bound the worst case at O(log n)). See [[08-quality-and-operations/security/hashing|Hashing (Security)]] and [[hash-functions]].
- **Mutating a key after insertion** changes its hash; the entry is now filed under the old bucket and becomes unreachable. Keys must be effectively immutable.
- **`% capacity` with a non-prime, non-power-of-two capacity** can amplify patterns in low-quality hashes; the modulus and hash must be chosen together.
- **Big-O hides cache behavior.** Chaining chases pointers across the heap (poor locality); open addressing stays in one contiguous array and is often faster in practice despite the same asymptotics — a recurring [[big-o-vs-reality]] theme.
- On disk, the equivalent lookup problem is usually solved by B-trees rather than hashes because hashing destroys the locality that block-oriented storage depends on; see [[04-databases/indexing|Database Indexing]].

## See Also
- [[hash-functions]] — computing the bucket index and defending it
- [[collision-resolution]] — chaining vs open addressing
- [[hash-maps-and-sets]] — the map/set ADTs built on this structure
- [[amortized-analysis]] — why doubling gives amortized O(1)
- [[08-quality-and-operations/security/hashing|Hashing (Security)]] — hash-flooding DoS
- [[04-databases/indexing|Database Indexing]] — hash indexes vs B-tree indexes
