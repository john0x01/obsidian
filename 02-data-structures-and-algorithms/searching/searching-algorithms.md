# Searching Algorithms

Searching is the problem of locating a key — or deciding it is absent — inside a collection, and the *achievable* cost is dictated entirely by what structure the data already has. This note maps the landscape so you can reach for the right technique: an unstructured bag forces a linear scan, sortedness unlocks logarithmic halving, and purpose-built structures (hash tables, trees, tries, graphs) let you buy a better bound up front at the cost of building and maintaining them.

## The Idea: Structure Dictates the Bound

The single most important lens is *how much the data lets you skip*. With no structure you must inspect every element in the worst case — `O(n)`. If the elements are **sorted**, each comparison eliminates half the remaining range, giving `O(log n)`. If you invest in a **hash table**, a good hash sends you to the answer in expected `O(1)`. If you build a **balanced tree** you keep `O(log n)` search while also supporting ordered operations (range, successor). The recurring trade is *preprocessing cost + space* traded against *query cost*. A single lookup rarely justifies sorting (`O(n log n)`) or building a hash map (`O(n)`); thousands of lookups against the same data almost always do.

Three families:

```text
UNORDERED         ORDERED (sorted array)      STRUCTURAL
linear scan       binary / jump /             BST, trie,
hash lookup       interpolation / exponential graph BFS/DFS
```

## The Two Poles in Code

Everything sits between these two extremes — scan everything, or halve the range:

```ts
// Unordered: inspect each element until found. Works on ANY array.
function linear(a: number[], x: number): number {
  for (let i = 0; i < a.length; i++) if (a[i] === x) return i;
  return -1;
}

// Ordered: requires a SORTED array; each step discards half the range.
function binary(a: number[], x: number): number {
  let lo = 0, hi = a.length;              // half-open [lo, hi)
  while (lo < hi) {
    const mid = lo + ((hi - lo) >> 1);
    if (a[mid] === x) return mid;
    if (a[mid] < x) lo = mid + 1;         // discard left half
    else hi = mid;                        // discard right half
  }
  return -1;
}
```

The difference is a precondition (`sorted`) buying an exponential improvement in the exponent's base — from `n` steps to `log₂ n`.

## Comparison Table

| Algorithm | Precondition | Time (avg / worst) | Extra space |
|---|---|---|---|
| [[linear-search]] | none | `O(n)` / `O(n)` | `O(1)` |
| Hash lookup ([[hash-tables]]) | hash + buckets built | `O(1)` / `O(n)` | `O(n)` |
| [[binary-search]] | sorted, random access | `O(log n)` / `O(log n)` | `O(1)` |
| [[jump-search]] | sorted | `Θ(√n)` / `Θ(√n)` | `O(1)` |
| [[interpolation-search]] | sorted, ~uniform | `O(log log n)` / `O(n)` | `O(1)` |
| [[exponential-search]] | sorted, unbounded ok | `O(log i)` / `O(log i)` | `O(1)` |
| [[ternary-search]] | sorted / unimodal | `O(log₃ n)` / `O(log₃ n)` | `O(1)` |
| [[binary-search-trees]] | balanced tree built | `O(log n)` / `O(n)`* | `O(n)` |
| [[tries]] | trie built | `O(L)` (key length) | `O(Σ·nodes)` |
| Graph [[graph-traversal]] | graph built | `O(V + E)` | `O(V)` |

*BST worst case is `O(n)` when unbalanced; self-balancing variants (AVL, red-black) restore `O(log n)`.

## Dry Run: Same Query, Two Families

Search `x = 23` in the sorted array `[4, 8, 15, 16, 23, 42]` (indices 0–5).

**Linear** scans left to right: `4≠23`, `8≠23`, `15≠23`, `16≠23`, `23=23` → found at index 4 after **5 comparisons**.

**Binary** on `[lo,hi) = [0,6)`:

| step | lo | hi | mid | a[mid] | action |
|---|---|---|---|---|---|
| 1 | 0 | 6 | 3 | 16 | `16 < 23` → `lo = 4` |
| 2 | 4 | 6 | 5 | 42 | `42 > 23` → `hi = 5` |
| 3 | 4 | 5 | 4 | 23 | found → return 4 |

Binary reaches the answer in **3 comparisons**; on `n = 6` the difference is small, but at `n = 10⁶` it is 20 versus a million — the exponential gap that motivates ordering.

## When Each Family Wins

- **Unordered (linear / hash).** Linear search is the only option on unsorted, unindexed, or singly-linked data, and it wins outright for tiny or one-shot inputs where preprocessing does not amortize. Hashing is the default for membership/lookup by exact key when order is irrelevant and you can afford the table — see [[hash-tables]].
- **Ordered (binary & friends).** Once data is sorted and randomly indexable, binary search and its cousins dominate. Pick the cousin by the *cost model*: [[jump-search]] when jumping backward is expensive (block/sequential storage), [[interpolation-search]] on near-uniform numeric keys, [[exponential-search]] when the size is unknown or the target is near the front.
- **Structural.** When you need *ordered* queries (range, successor, min/max) alongside lookup, a [[binary-search-trees|BST]] beats a flat array because inserts stay `O(log n)`; a flat sorted array costs `O(n)` per insert. [[tries]] search by key *prefix* in `O(L)` independent of how many keys are stored — ideal for autocomplete and routing. Searching a relationship graph (reachability, shortest path) is a traversal problem: [[graph-traversal]] via BFS/DFS, and [[tree-traversals]] for hierarchical data.

## Interview Pitfalls & Gotchas

- **Sorting to enable binary search, then searching once.** The sort is `O(n log n)`; a single lookup is cheaper with a linear scan. Sort only when many queries share it.
- **Assuming `O(1)` hash lookups are free.** They cost table build time and space, degrade to `O(n)` under adversarial collisions, and lose ordered queries entirely.
- **Forgetting the sorted precondition.** Binary/jump/interpolation search silently return garbage on unsorted input; they cannot detect the violation.
- **Ignoring [[07-performance-engineering/big-o-in-practice|real hardware]].** For small `n`, a cache-friendly linear scan beats binary search's non-local jumps despite the worse Big-O — a classic [[big-o-vs-reality]] case.

## See Also

- [[linear-search]]
- [[binary-search]]
- [[hash-tables]]
- [[binary-search-trees]]
- [[tries]]
- [[graph-traversal]]
- [[tree-traversals]]
