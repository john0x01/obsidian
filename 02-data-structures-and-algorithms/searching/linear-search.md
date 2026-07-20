# Linear Search

Linear search — also called sequential search — walks the collection from one end to the other, comparing each element to the target until it finds a match or runs off the end. It is the humblest algorithm in the toolbox and the *only* one that works with **no precondition at all**: no sorting, no index, no hashing. Reach for it on unsorted data, on linked structures you cannot randomly index, and on inputs small enough that anything cleverer is not worth the trouble.

## The Idea

Inspect elements one at a time in order; return the position of the first match, or a sentinel (`-1`) if the scan completes without one. There is no invariant to preserve beyond "everything before the cursor has been checked and did not match."

```text
target = 23
[ 4 | 8 | 15 | 16 | 23 | 42 ]
  ✗   ✗   ✗    ✗    ✓        ← stop at index 4
```

## Implementation

```ts
function linearSearch<T>(a: T[], x: T): number {
  for (let i = 0; i < a.length; i++) {
    if (a[i] === x) return i;   // first match wins
  }
  return -1;                    // absent
}
```

That is the whole algorithm. A variant returns *all* positions, or takes a predicate (`a.findIndex` in JS is exactly this). One classic micro-optimization is the **sentinel** trick: append the target to the end so the match is guaranteed, which lets you drop the `i < a.length` bounds check from the hot loop and test only `a[i] === x`:

```ts
function sentinelSearch(a: number[], x: number): number {
  const n = a.length;
  const last = a[n - 1];
  a[n - 1] = x;                 // plant a guaranteed match at the end
  let i = 0;
  while (a[i] !== x) i++;       // no bounds check — the sentinel stops us
  a[n - 1] = last;              // restore
  if (i < n - 1 || last === x) return i;
  return -1;
}
```

The sentinel halves the number of comparisons *per iteration* (one test instead of two), a constant-factor win that mattered on older hardware; modern branch predictors and JITs blunt it, and it mutates the array, so it is more a piece of algorithmic folklore than a go-to. Know it for interviews.

## Dry Run

Search `x = 23` in `[4, 8, 15, 16, 23, 42]`:

| i | a[i] | a[i] === 23? |
|---|---|---|
| 0 | 4 | no |
| 1 | 8 | no |
| 2 | 15 | no |
| 3 | 16 | no |
| 4 | 23 | **yes → return 4** |

Five comparisons. Now search for `x = 99` (absent): the loop runs all six iterations, every test fails, and it returns `-1` — the worst case touches every element.

## Complexity

- **Best case `O(1)`** — the target is the first element, one comparison.
- **Average case `O(n)`** — assuming the target is present and uniformly likely at any position, the expected number of comparisons is `(n+1)/2`, which is `Θ(n)`. If the target may be absent, an unsuccessful search always costs exactly `n` comparisons.
- **Worst case `O(n)`** — target last or absent; every element is inspected.
- **Space `O(1)`** — a single index variable, iterative, no recursion.

The bound is `Θ(n)` because the algorithm extracts no information from the data's arrangement: each comparison rules out exactly one element, so eliminating `n` candidates takes `n` comparisons. Contrast [[binary-search]], where each comparison rules out *half* the candidates — that is the entire reason it reaches `O(log n)`. Linear search cannot be improved without adding structure (sorting, hashing, indexing).

## Variants & Trade-offs

- **`findIndex` / `indexOf`** — the standard-library incarnation; `indexOf` for primitive equality, `findIndex` for a predicate.
- **Move-to-front / transpose heuristics** — on repeated searches with skewed access frequency, moving a found element toward the front makes hot keys cheaper over time (self-organizing lists). Useful when a few keys dominate queries.
- **Sentinel search** — the constant-factor trim above.

## When to Use — and When It Beats Binary Search

Use linear search whenever the data is **unsorted or unindexed** (its only real competitor there is building a hash set, which costs `O(n)` space and time up front). But it also wins in situations where binary search is *available yet slower in practice*:

- **Tiny arrays.** For `n` below roughly 8–64 elements, a linear scan's tight, branch-predictable loop and sequential memory access beat binary search's non-local jumps and branch mispredicts. This is exactly why production sorts (introsort, Timsort) cut over to insertion sort — built on linear scanning — for small subarrays. See [[big-o-vs-reality]] and [[07-performance-engineering/big-o-in-practice|Big-O in Practice]].
- **Linked lists.** Binary search needs `O(1)` random access to compute `a[mid]`; a linked list gives you only `O(n)` positional access, which destroys the halving advantage. Sequential traversal is the natural — and only sensible — search here.
- **Cache-friendly contiguous scans.** A forward scan over a contiguous array reads memory in the exact order the prefetcher expects, so it sustains near-peak bandwidth. For a *one-shot* search, this locality (plus zero preprocessing) often makes linear search the fastest real-world option even when `n` is not tiny.
- **One-shot queries on large data.** If you will search once, paying `O(n log n)` to sort first is strictly worse than a single `O(n)` scan.

## Interview Pitfalls & Gotchas

- **Reaching for it on sorted data.** If the input is already sorted and randomly indexable, an `O(n)` scan wastes the ordering — use [[binary-search]].
- **Reference vs value equality.** `===` compares object references in JS; two structurally-equal objects will not match. Use a predicate (`findIndex`) with a proper comparison for object arrays.
- **Off-by-one at the boundary** in the sentinel variant — distinguishing "found at the real last index" from "found the planted sentinel" is the subtle bug; the restore-and-recheck step is mandatory.
- **Assuming `O(n)` is always slow.** For small or unsorted inputs it is frequently the *fastest* choice; do not over-engineer a hash set or a sort for a handful of lookups.
- **Not returning early.** Continuing the loop after the first match wastes work when only the first position is needed.

## See Also

- [[binary-search]]
- [[searching-algorithms]]
- [[hash-tables]]
- [[big-o-vs-reality]]
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]]
