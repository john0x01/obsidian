# Segment Trees & Fenwick Trees

Both structures answer **range queries over an array that also changes** — "sum/min/max of `a[l..r]`" interleaved with updates. A static [[prefix-sums|prefix-sum]] array answers range-sum in O(1) but costs O(n) to rebuild after any update; segment trees and Fenwick (Binary Indexed) trees give **O(log n) for *both* query and update**, which is the whole point.

## Segment Tree

Build a binary tree over the array: each leaf is one element, and each internal node stores the **merge** of its two children over the sub-range it covers. Merge can be any *associative* operation — sum, min, max, gcd, matrix product — which makes segment trees the general tool.

```
         [0..7] sum=36
        /            \
   [0..3]=10       [4..7]=26
   /     \          /     \
 ...     ...      ...     ...
```

- **Build:** O(n) — one merge per node, and there are ~2n nodes.
- **Point update:** O(log n) — change a leaf, walk to the root re-merging each ancestor.
- **Range query:** O(log n) — the range `[l..r]` decomposes into at most O(log n) *canonical* nodes whose union is exactly the range; combine them.

A common iterative implementation uses a `2n` array (leaves in the second half); recursive versions allocate `4n` to be safe for non-power-of-two sizes.

```ts
// point update + range-sum, iterative, 1-based leaves at [n, 2n)
function update(t: number[], n: number, i: number, v: number) {
  for (t[i += n] = v; i > 1; i >>= 1) t[i >> 1] = t[i] + t[i ^ 1];
}
```

### Lazy Propagation

A naive segment tree can't do a **range update** ("add x to every element in `[l..r]`") faster than O(n). **Lazy propagation** fixes this: when an update covers a whole node's range, apply it to that node and stash a pending "lazy" tag instead of recursing into children. The tag is *pushed down* only when a later query or update must descend through the node. This makes **range update + range query both O(log n)**. Getting the push-down and tag-composition right (especially for assign-vs-add or min-with-add) is the classic source of subtle bugs.

## Fenwick / Binary Indexed Tree

A Fenwick tree computes **prefix sums with point updates in O(log n)** using a single array and no explicit tree — the structure is *implicit in the binary representation of the index*. Index `i` (1-based) is responsible for a range of length equal to its **lowest set bit**, `i & -i`, ending at `i`. `i & -i` isolates that bit via two's-complement negation.

```ts
function add(bit: number[], n: number, i: number, delta: number) {
  for (; i <= n; i += i & -i) bit[i] += delta;   // climb responsibility ranges
}
function prefix(bit: number[], i: number): number {
  let s = 0;
  for (; i > 0; i -= i & -i) s += bit[i];         // peel off ranges
  return s;
}
```

A range sum `[l..r]` is `prefix(r) − prefix(l−1)`. Both loops run once per set bit, so **O(log n)**.

### Trade-offs vs Segment Tree

- **Memory & constants:** the BIT is one array of size n+1 — roughly half the memory of a segment tree, with tighter loops and better cache behavior. When it fits, it's the faster choice for sum-style problems.
- **Generality:** it's harder to use for arbitrary associative operations. Prefix sums rely on an **invertible** operation (subtract to get an arbitrary range from two prefixes). For non-invertible operations like **range min/max**, a BIT can only answer *prefix* min cleanly — arbitrary range min needs extra machinery, whereas a segment tree handles any associative merge directly. This is the core reason to reach for a segment tree.
- **Range updates:** achievable with one or two BITs (the range-update/range-query trick), but a lazy segment tree expresses richer updates more naturally.

## Choosing, and vs Static Prefix Sums

- **Immutable array, sum queries:** use plain [[prefix-sums]] — O(1) query, no beating it.
- **Mutable array, sum/point ops:** Fenwick tree — smallest and fastest.
- **Mutable array, non-invertible ops (min/max/gcd), range updates, or anything exotic:** segment tree, with lazy propagation if updates are ranges.

The segment tree is a [[divide-and-conquer]] decomposition made persistent in memory; the O(log n) bounds come from the tree height, so review [[recurrence-relations]] and [[big-o-vs-reality]] if the constants surprise you.

## Senior Pitfalls

- **Off-by-one and 1- vs 0-indexing:** Fenwick math assumes 1-based indices; `prefix(0)` must be the identity. Mixing conventions silently corrupts results.
- **Overflow:** range sums over large arrays overflow 32-bit ints — use `BigInt` or 64-bit where the language allows.
- **Lazy tag composition:** combining two pending updates (e.g. an assignment then an add) is order-sensitive; get the algebra wrong and queries drift.
- **Non-associative merges don't work** — segment trees require associativity; a "median over range" is not a segment-tree operation without heavier structures.

## See Also
- [[prefix-sums]] — the O(1)-query static baseline
- [[binary-trees]] — the tree shape a segment tree is built on
- [[bit-manipulation]] — `i & -i` and the Fenwick indexing trick
- [[divide-and-conquer]] — the range-decomposition principle
- [[recurrence-relations]] — where the O(log n) height comes from
