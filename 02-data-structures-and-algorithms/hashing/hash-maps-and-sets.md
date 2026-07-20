# Hash Maps & Sets

Hash maps and hash sets are the two everyday abstract data types built on a [[hash-tables|hash table]]. A **map** (dictionary/associative array) stores key→value associations; a **set** stores a collection of distinct keys with no values. They are the same machinery viewed through two ADTs, and knowing where the abstraction leaks — ordering, iteration, complexity — is what separates using them from using them correctly.

## The Two ADTs

- **Map ADT**: `set(k, v)`, `get(k) → v | absent`, `has(k) → bool`, `delete(k)`. Keys are unique; each maps to exactly one value.
- **Set ADT**: `add(k)`, `has(k) → bool`, `delete(k)`, plus algebraic operations — union, intersection, difference.

A set is literally *a map whose values carry no information* — a "map to unit." Many languages implement it exactly that way: Java's `HashSet` wraps a `HashMap` with a shared dummy value object. So everything true of a hash map's performance and ordering is true of the corresponding hash set.

Two generalizations relax the uniqueness constraint:

- **Multiset (bag)** — allows duplicate keys; implemented as a `Map<K, count>`. The idiomatic counting pattern (frequency tables, [[sliding-window]] problems) is exactly a multiset.
- **Multimap** — one key to many values; a `Map<K, V[]>` or `Map<K, Set<V>>`.

## Ordered vs Unordered

This is the property that trips people up. A hash-based map/set is **unordered**: iteration order is dictated by bucket layout and can change on resize (see [[collision-resolution]]). If you need order you have two different things:

- **Insertion order** — remember the sequence keys were added.
- **Sorted order** — keep keys in comparison order, enabling range queries and predecessor/successor.

Sorted maps/sets are *not* hash-based at all; they are backed by a balanced [[binary-search-trees|BST]] ([[red-black-trees]]), trading O(1) hashing for O(log n) comparisons in exchange for order and range queries. Reaching for a "sorted hash map" is a category error.

## Complexity

For all hash-backed variants, `get`/`set`/`has`/`add`/`delete` are **O(1) average, O(n) worst** — the same bounds and the same simple-uniform-hashing assumption as the underlying [[hash-tables|hash table]]. Insertion is **amortized O(1)** across resizes (see [[amortized-analysis]]). Set operations are proportional to the smaller operand: union/intersection of sets sized a and b is O(a + b) average — you iterate one and probe the other, so always iterate the *smaller* set. Space is O(n). Tree-backed sorted variants are O(log n) per operation, worst-case, with ordered iteration for free.

## Language Implementations & Their Semantics

The dangerous assumption is that all these types behave alike. They differ precisely on ordering and iteration:

```ts
// JS Map/Set guarantee INSERTION-order iteration (spec-mandated).
const m = new Map<string, number>();
m.set("b", 1).set("a", 2);
[...m.keys()];          // ["b", "a"] — insertion order, NOT sorted
```

- **JavaScript `Map` / `Set`** — hash-based but **spec-guaranteed to iterate in insertion order** (integer-like keys in `Map` are a subtle exception, ordered ascending first). Distinct from a plain object `{}`, which coerces keys to strings and has murkier ordering rules; prefer `Map` for non-string keys and predictable iteration. All ops O(1) average.
- **Java `HashMap` / `HashSet`** — hash-based, **no ordering guarantee**; iteration order is arbitrary and may change on resize. Treeifies overloaded buckets to O(log n) (Java 8+). Permits one `null` key.
- **Java `LinkedHashMap` / `LinkedHashSet`** — hash table plus a doubly-linked list threading entries, giving **insertion-order** (or access-order) iteration at a small memory cost. The access-order mode is the standard base for an [[lru-cache|LRU cache]].
- **Java `TreeMap` / `TreeSet`** — **red-black tree**, not a hash table: **sorted** iteration, O(log n) ops, and range views (`headMap`, `subMap`, `ceilingKey`). Choose it when you need order or range, accept the log factor.

The senior takeaway: **when your algorithm's correctness depends on iteration order, never use a bare `HashMap`/unordered set** — pick the insertion-ordered or sorted variant explicitly.

## Canonical Patterns

- **Frequency counting / anagrams / dedup** — multiset via `Map<K, count>`; set for "seen" tracking in two-pointer and [[sliding-window]] problems.
- **Indexing / joins** — build a hash index on one side, probe with the other (the hash-join primitive; cf. [[04-databases/indexing|Database Indexing]]).
- **Memoization** — cache from argument tuple to result (see [[07-performance-engineering/memoization|Memoization]]).
- **Graph adjacency / visited sets** — set membership drives BFS/DFS bookkeeping.

## Senior Pitfalls

- **Using objects/arrays as JS `Map` keys** compares by *reference*, not value — two equal-looking objects are distinct keys. Serialize to a primitive key, or you get silent misses.
- **Mutating a key after insertion** breaks its hash/equality contract and orphans the entry (Java: override `hashCode` *and* `equals` consistently, or lookups fail).
- **Relying on `HashMap` iteration order** for output is a classic non-deterministic bug that passes locally and fails in CI.
- **`for...in` over a JS object** to emulate a map walks the prototype chain and stringifies keys — use `Map`.

## See Also
- [[hash-tables]] — the structure both ADTs sit on
- [[hash-functions]] — key hashing and the equals/hashCode contract
- [[collision-resolution]] — why unordered iteration is unordered
- [[binary-search-trees]] — ordered map/set alternative
- [[red-black-trees]] — what backs `TreeMap`/`TreeSet`
- [[lru-cache]] — `LinkedHashMap` in access-order mode
