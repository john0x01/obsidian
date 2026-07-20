# Bloom Filters

A Bloom filter is a compact, probabilistic set that answers one question — "have I *possibly* seen this element?" — in constant time and a few bits per element. It trades exactness for space: it can produce **false positives** but **never false negatives**. That one-sided error is the whole point: a "no" is definitive, a "yes" means "probably." It solves the problem of testing membership against a huge set when storing the actual keys would be prohibitively large.

## The Idea: One Bit Array, k Hash Functions

The structure is a bit array of *m* bits, all initially 0, plus *k* independent [[hash-functions|hash functions]], each mapping an element to one of the *m* positions.

- **Insert(x):** compute the *k* hashes and set all *k* bits to 1.
- **Query(x):** compute the same *k* hashes; if **all** *k* bits are 1, answer "possibly present"; if **any** is 0, answer "definitely absent."

```ts
function add(x: T): void {
  for (let i = 0; i < k; i++) bits[hash(x, i) % m] = 1;   // seed i => independent hash
}
function mightContain(x: T): boolean {
  for (let i = 0; i < k; i++) if (bits[hash(x, i) % m] === 0) return false; // no false negatives
  return true;                                                             // maybe — could collide
}
```

**Why no false negatives:** if *x* was ever inserted, its *k* bits were set and are never cleared, so a query for *x* always finds them all set. **Why false positives happen:** *x*'s *k* bits may all have been set incidentally by *other* elements, so "all bits set" doesn't prove *x* was inserted. In practice *k* is generated cheaply from two base hashes via `g_i(x) = h1(x) + i·h2(x)` (Kirsch–Mitzenmacher double hashing) rather than *k* separate functions.

## The Space/Accuracy Trade-off and the Formulas

With *n* inserted elements, *m* bits, and *k* hash functions, the probability a given bit is still 0 is (1 − 1/m)^{kn} ≈ e^{−kn/m}. A false positive needs all *k* queried bits set, so:

```
false-positive rate  p ≈ (1 − e^(−kn/m))^k
```

For fixed *m* and *n*, this is minimized at the **optimal number of hashes**:

```
k* = (m / n) · ln 2                (round to nearest integer)
```

Substituting back gives the clean relations at optimal *k*:

```
p = (1/2)^k*        i.e.   each added hash halves the FP rate
m / n = − ln p / (ln 2)²  ≈  −1.44 · log₂ p     (bits needed per element)
```

Concretely: **≈ 9.6 bits/element for p = 1%** (k ≈ 7), **≈ 14.4 bits/element for p = 0.1%** — orders of magnitude smaller than storing the keys. The FP rate depends on the *ratio* m/n, not absolute size. Operations are **O(k) time** — independent of *n* — and **O(m) space**.

The senior insight in the formula: too few hashes underuses the bits; too many saturate the array to all-1s and *raise* the FP rate. There is a genuine optimum, and it assumes you know *n* in advance — overshoot *n* and *p* degrades sharply.

## The Deletion Problem and Counting Bloom Filters

You **cannot delete** from a standard Bloom filter: clearing an element's *k* bits might clear a bit shared with another element, creating a false *negative* — which breaks the core guarantee. The fix is the **Counting Bloom Filter**: replace each bit with a small counter (e.g. 4 bits); insert increments the *k* counters, delete decrements them. This restores deletion at ~4× the space and reintroduces a tiny risk of counter overflow. **Cuckoo filters** and **quotient filters** are modern alternatives that support deletion with better locality and comparable space.

## When To Use It & Applications

The killer use is **read avoidance**: place a cheap filter in front of an expensive lookup so that the common "not present" case skips the expensive path entirely.

- **[[04-databases/lsm-trees|LSM-trees]]:** each on-disk SSTable carries a Bloom filter; a point read consults the filter first and skips reading the table's blocks from disk when the filter says "absent" — turning most negative lookups into a few in-memory bit tests. This is the single most important Bloom application in databases and pairs with [[04-databases/indexing|indexing]].
- **Caches / CDNs:** "cache one-hit-wonders" — only admit an object to cache on its *second* request, tracked cheaply by a Bloom filter, to keep single-access junk out.
- **Deduplication & crawlers:** "have I seen this URL / this event?" without storing every key.
- **Distributed systems:** compact set digests to avoid transferring or querying remote data.

## Senior Pitfalls & Misconceptions

- **A "yes" is never proof of presence** — always confirm against the source of truth if a false positive is costly. Bloom filters only make the *negative* answer authoritative.
- **You must size for the true *n*.** The FP rate assumes a known element count; filling past it silently degrades accuracy. For streaming/unknown *n*, use a **Scalable Bloom Filter** (chained filters of growing size).
- **Not adversary-proof.** With a known, unkeyed hash, an attacker can craft inputs that all collide into the same bits to inflate the FP rate — use a keyed/seeded hash where inputs are untrusted (see [[08-quality-and-operations/security/hashing|Hashing (Security)]]).
- **Poor cache locality:** *k* random probes scatter across the bit array, so each query is up to *k* cache misses; *blocked* Bloom filters confine an element's bits to one cache line to fix this. (Union/intersect of two filters is only valid when they share identical *m*, *k*, and seeds.)

## See Also
- [[hash-functions]] — the k independent hashes and double-hashing trick
- [[04-databases/lsm-trees|LSM-Trees]] — Bloom filters as SSTable read-avoidance
- [[04-databases/indexing|Database Indexing]] — where membership filters sit in the read path
- [[08-quality-and-operations/security/hashing|Hashing (Security)]] — keyed hashes against adversarial inputs
- [[hash-tables]] — the exact-membership structure Bloom filters compress
