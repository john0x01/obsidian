# Quickselect & Top-K

"Find the k-th smallest element" or "return the K largest" does **not** require a full sort. Two families solve it: **quickselect** (partition and recurse into only one side — expected linear) and a **size-K heap** (stream elements past a bounded heap — O(n log k)). The right choice hinges on whether you can mutate the array once, or must consume a stream / preserve the input.

## Quickselect — the idea

Quicksort partitions around a pivot so that everything left of the pivot is ≤ it and everything right is ≥ it; the pivot lands at its **final sorted position** `p`. Quickselect exploits that: if you only want rank `k`, you never recurse into *both* halves. Compare `p` to `k` and descend into the single side that contains the answer.

```
want k=2 (0-indexed) in [7, 2, 9, 4, 5]
partition on pivot 5 → [4, 2, 5, 9, 7], pivot lands at p=2
p == k → answer is 5. The [9,7] side is never touched.
```

## Implementation

Lomuto partition with a random pivot (randomization is what keeps the average case honest):

```ts
function quickselect(a: number[], k: number): number { // k is 0-indexed rank
  let lo = 0, hi = a.length - 1;
  while (lo < hi) {
    const p = partition(a, lo, hi);
    if (p === k) return a[k];
    else if (p < k) lo = p + 1; // answer is to the right
    else hi = p - 1;            // answer is to the left
  }
  return a[lo];
}

function partition(a: number[], lo: number, hi: number): number {
  const r = lo + Math.floor(Math.random() * (hi - lo + 1));
  [a[r], a[hi]] = [a[hi], a[r]]; // random pivot → move to end
  const pivot = a[hi];
  let i = lo;
  for (let j = lo; j < hi; j++) {
    if (a[j] < pivot) { [a[i], a[j]] = [a[j], a[i]]; i++; }
  }
  [a[i], a[hi]] = [a[hi], a[i]]; // pivot into its final slot
  return i;
}
```

Written iteratively (tail-recursion eliminated) so it uses O(1) stack.

## Dry run / trace

`quickselect([7,2,9,4,5], k=2)`, suppose pivot 5 is chosen:

| pass | array (region) | pivot | `p` | vs `k=2` | next region |
|------|----------------|-------|-----|----------|-------------|
| 1 | `[7,2,9,4,5]` | 5 | 2 | `p == k` | **return `a[2]` = 5** |

Had `p` been 3, we'd set `hi = p−1 = 2` and re-partition only `[lo..2]` — the right side is discarded untouched.

## Complexity — why expected O(n)

Each partition is O(size). A balanced split leaves half the array, giving the recurrence **T(n) = T(n/2) + O(n) = O(n)** (a geometric series `n + n/2 + n/4 + … = 2n`), *not* the O(n log n) of sorting, because we recurse into **one** side, not both.

- **Best / average**: **O(n)** time, O(1) extra space (in place).
- **Worst**: **O(n²)** — an adversary (or already-sorted input with a naive pivot) forces each partition to peel off one element. Random pivots make this astronomically unlikely; **expected** O(n) holds regardless of input.

## Median-of-medians — guaranteed O(n)

To kill the O(n²) worst case deterministically, choose the pivot with **median-of-medians (BFPRT)**: split into groups of 5, find each group's median, then recursively select the median of those medians. That pivot guarantees at least ~30% of elements fall on each side, bounding the recursion. The recurrence `T(n) = T(n/5) + T(7n/10) + O(n)` solves to **O(n) worst case**. In practice the constant factor is large, so random-pivot quickselect is usually faster; median-of-medians matters when worst-case guarantees are contractual.

## The heap approach — streaming top-K

To keep the **K largest**, maintain a **min-heap of size K**. Each new element: if the heap has fewer than K, push it; otherwise if it exceeds the heap's minimum (root), replace the root. The root is always the k-th largest seen so far; the heap holds the winners.

```ts
function topK(nums: number[], k: number): number[] {
  const heap = new MinHeap(); // size capped at k
  for (const x of nums) {
    if (heap.size < k) heap.push(x);
    else if (x > heap.peek()) { heap.pop(); heap.push(x); }
  }
  return heap.toArray(); // the k largest (unordered)
}
```

- **Time O(n log k)**: `n` elements, each heap op O(log k).
- **Space O(k)**: only the survivors are stored — independent of `n`.

Use a **max-heap of size K** symmetrically for the K smallest.

## When to pick which

| | Quickselect | Size-K heap |
|---|---|---|
| Time | expected **O(n)** | **O(n log k)** |
| Space | O(1) in place | **O(k)** |
| Needs all data in memory? | yes (random access) | no — **streaming** |
| Mutates / reorders input? | yes | no — preserves order |
| Worst case | O(n²) (random) or O(n) (MoM) | O(n log k) always |

- **Quickselect** wins for a one-shot query on a mutable in-memory array when you only need *the* k-th element — expected linear and O(1) space.
- **Heap** wins when data **streams** (can't hold all `n`), when `k ≪ n` (log k tiny), when the input must stay **immutable**, or when you want the top-K *maintained incrementally* as new data arrives.
- If you need the top-K **sorted**, add an O(k log k) sort at the end (still cheaper than sorting all `n`).

## Interview pitfalls & gotchas

- **1-indexed vs 0-indexed k**: "3rd smallest" is rank `k−1` in a 0-indexed partition. Off-by-one here is the number-one bug.
- **k-th *largest* via quickselect**: it's the `(n − k)`-th smallest — convert the index, don't rewrite the comparator carelessly.
- **Non-random pivot → O(n²)**: choosing `a[lo]` or `a[hi]` on sorted/reverse-sorted input is the adversarial trap. Randomize or use median-of-3/median-of-medians.
- **Wrong heap polarity**: top-K largest needs a **min**-heap (you evict the smallest survivor); people instinctively reach for a max-heap and get it backwards.
- **Duplicates**: quickselect handles them, but Lomuto with many equal keys degrades toward O(n²); three-way (Dutch-flag) partitioning fixes it.
- **Destroying the caller's array**: quickselect reorders in place — copy first if the caller still needs the original order.

## See Also

- [[quick-sort]] — quickselect is quicksort that recurses into one side
- [[heaps]] — the size-K heap engine
- [[priority-queues]] — the streaming top-K abstraction
- [[sorting-algorithms]] — the O(n log n) baseline both approaches beat
- [[divide-and-conquer]] — the recurrence framing behind expected O(n)
