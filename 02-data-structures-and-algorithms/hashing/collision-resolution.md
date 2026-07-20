# Collision Resolution

Because a [[hash-functions|hash function]] maps a huge key space into a small bucket array, two distinct keys will inevitably share a bucket — a *collision*. Collision resolution is the strategy a [[hash-tables|hash table]] uses to store and later find both, and it is the single biggest determinant of the table's real-world performance.

## Separate Chaining

Each bucket holds a secondary container — classically a linked list — of all entries that hashed there.

```
bucket[3] → (key=k1,v) → (key=k7,v) → null
```

Insert prepends (O(1)); lookup hashes to the bucket then scans the chain, comparing keys with `===`/`.equals`. Expected chain length is the **load factor** α = n/m, so search is O(1 + α). Chaining tolerates α > 1 gracefully and never "fills up," which makes deletion trivial (just unlink). Its costs: a pointer per node (memory overhead), and poor **cache locality** — each hop chases a pointer to a random heap address, so a long chain thrashes the cache even though its Big-O looks fine. Modern implementations (Java 8+ `HashMap`) mitigate the O(n) worst case by **treeifying** a bucket into a red-black tree once its chain exceeds ~8, bounding it at O(log n).

## Open Addressing

Store every entry *inside the array itself*; on collision, follow a deterministic **probe sequence** to the next candidate slot. There are no side lists, so memory is compact and access is cache-friendly — but the array must keep α < 1, and a full table cannot accept inserts.

- **Linear probing** — try `h, h+1, h+2, …` (mod m). Maximally cache-friendly (sequential memory), but suffers **primary clustering**: occupied slots coalesce into long runs, and any key hashing anywhere into a run must traverse the whole thing. Clusters grow super-linearly as α rises.
- **Quadratic probing** — try `h + 1², h + 2², …`. Breaks up primary clusters, but keys sharing the *same initial bucket* still follow the identical probe sequence — **secondary clustering** — and the probe sequence may not cover all slots unless m and the constants are chosen carefully (a power-of-two table with triangular numbers `h + i(i+1)/2` provably visits every slot).
- **Double hashing** — step by a second hash: `h1(k) + i·h2(k)`. Different keys get different strides, eliminating both clustering forms and giving the best distribution, at the cost of computing a second hash (weaker locality than linear probing). `h2` must never return 0 and should be coprime to m.

### Deletion and Tombstones

Deletion is the subtle part of open addressing. You cannot simply empty a slot: a lookup probing *past* that slot would hit the gap and wrongly conclude the key is absent, breaking the probe chain. The fix is a **tombstone** — a "deleted" marker that lookups probe *through* but inserts may overwrite. Tombstones accumulate and slowly degrade search (they count against the probe length though not against α), so heavily-churned tables periodically rehash to purge them.

## Load-Factor Thresholds

- **Chaining** stays viable well past α = 1; typical resize trigger ~0.75 balances space and chain length.
- **Open addressing** degrades sharply as α → 1 (probe count for linear probing ≈ ½(1 + 1/(1−α)²)), so it resizes earlier, commonly at α ≈ 0.5–0.7.

Crossing the threshold triggers a resize + **rehash** — an O(n) rebuild amortized to O(1) per insert (see [[amortized-analysis]]).

## Refinements: Robin Hood and Hopscotch

- **Robin Hood hashing** (open addressing) tracks each entry's *probe distance from its home bucket* and, on insert, "steals from the rich": if the incoming key has probed farther than the resident, they swap. This equalizes probe distances, slashing variance so the *worst* lookup stays short even at high α — the property that makes it popular in Rust's older `HashMap` and in Swiss tables.
- **Hopscotch hashing** guarantees an entry lives within a small neighborhood (H slots) of its home bucket, so lookups scan a fixed, cache-line-sized window; inserts displace entries to keep the invariant. Excellent locality and concurrency behavior.

## Trade-off Summary

| | Chaining | Open addressing |
|---|---|---|
| Memory | +pointer/node | compact, in-array |
| Locality | poor (pointer chasing) | excellent (contiguous) |
| Max load factor | > 1 fine | must stay < 1 |
| Deletion | trivial unlink | needs tombstones |
| Worst case | O(n) (O(log n) if treeified) | O(n) under clustering |
| Best for | unknown/large entries, adversarial input | small entries, cache-sensitive, known bounds |

## Senior Pitfalls

- **Deleting by clearing a slot** in open addressing silently corrupts lookups — always tombstone or backward-shift.
- **Underestimating clustering:** linear probing's average looks fine until α climbs, then latency explodes non-linearly.
- **Chaining's cache cost** is invisible to Big-O; on modern CPUs a compact open-addressed table often beats a chained one with identical asymptotics — a classic [[big-o-vs-reality]] / [[07-performance-engineering/data-locality|Data Locality]] lesson.
- Long adversarial chains are the substrate of hash-flooding; pair a good scheme with a keyed hash (see [[hash-functions]]).

## See Also
- [[hash-tables]] — the structure being resolved
- [[hash-functions]] — collision rate depends on hash quality
- [[amortized-analysis]] — the resize/rehash cost model
- [[07-performance-engineering/data-locality|Data Locality]] — why chaining's Big-O misleads
- [[big-o-vs-reality]] — asymptotics vs measured cost
