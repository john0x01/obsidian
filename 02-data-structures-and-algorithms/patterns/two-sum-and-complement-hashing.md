# Two-Sum & Complement Hashing

Complement hashing is the reflex for "find a pair (or group) that combines to a target" over an unordered collection: instead of trying every pair in O(n²), you remember what you have seen and, for each new element `x`, ask a single O(1) question — *have I already seen `target − x`?* When you must return the pair itself (indices or values), a hash map answers in one pass.

## The idea / invariant

Maintain a map `seen` of every value encountered so far → its index. When you reach `x`, its **complement** is `need = target − x`. If `need` is already a key in `seen`, you have found a valid pair *before you ever revisited an element*. The invariant: **by the time you process index `i`, `seen` contains exactly the elements at indices `0..i−1`.** So a hit at `i` pairs `x` with an earlier element — no pair is missed and none is double-counted.

```
target = 9,  nums = [2, 7, 11, 15]
i=0 x=2  need=7  seen={}          miss  → seen={2:0}
i=1 x=7  need=2  seen={2:0}       HIT   → return [0, 1]
```

## Implementation

The canonical one-pass complement lookup, returning the two indices:

```ts
function twoSum(nums: number[], target: number): [number, number] | null {
  const seen = new Map<number, number>(); // value -> index
  for (let i = 0; i < nums.length; i++) {
    const need = target - nums[i];
    if (seen.has(need)) {
      return [seen.get(need)!, i]; // earlier index first
    }
    seen.set(nums[i], i); // record AFTER checking, so x can't pair with itself
  }
  return null;
}
```

The ordering matters: check first, then insert. Inserting before the check would let a single element with `x === need` (i.e. `2x === target`) match itself.

## Dry run / trace

`twoSum([3, 2, 4], 6)`:

| i | x | need = 6−x | `seen` before | action |
|---|---|-----------|---------------|--------|
| 0 | 3 | 3 | `{}` | miss → `{3:0}` |
| 1 | 2 | 4 | `{3:0}` | miss → `{3:0, 2:1}` |
| 2 | 4 | 2 | `{3:0, 2:1}` | **HIT** on 2 → return `[1, 2]` |

Note that index 0 (value 3) is *never* wrongly paired with itself even though `3 + 3 = 6`, because we only stored one copy and checked before inserting.

## Complexity

- **Time O(n)**: one pass; each `has`/`get`/`set` is expected O(1) on a hash map. Worst case O(n) per op under pathological collisions, but treated as O(1) amortized.
- **Space O(n)**: the map can grow to hold all `n` elements if no pair is found.

This trades the naive brute force's O(1) space for a linear speedup — the classic space-for-time deal.

## The sorted two-pointer alternative

If the array is (or can be) sorted and you only need *values* or a boolean, [[two-pointer]] beats hashing on space. Put `lo` at the start, `hi` at the end; the sum is monotonic in the window:

```ts
function twoSumSorted(a: number[], target: number): [number, number] | null {
  let lo = 0, hi = a.length - 1;
  while (lo < hi) {
    const sum = a[lo] + a[hi];
    if (sum === target) return [lo, hi];
    if (sum < target) lo++;   // need bigger → advance left
    else hi--;                // need smaller → retreat right
  }
  return null;
}
```

- Already sorted: **O(n) time, O(1) extra space** — strictly better than hashing on memory.
- Must sort first: **O(n log n) time**, but still O(1) extra (or O(log n) stack). The catch: sorting **destroys original indices**, so if the problem demands the *positions* in the input, either pair values with their original index before sorting, or use the hash approach.

## Generalization to 3-sum and k-sum

Fix the outer element(s) and reduce to two-sum-on-a-sorted-array. **3-sum**: sort, then for each `i` run a two-pointer over `i+1..n−1` looking for `−nums[i]`. That is O(n) outer × O(n) inner = **O(n²)**. **k-sum** recurses: fix one element, solve (k−1)-sum on the rest, bottoming out at the two-pointer base case → **O(n^{k−1})** time (plus the initial sort). Sorting is what makes both duplicate-skipping and the two-pointer sweep possible.

Duplicate handling in 3-sum: after sorting, skip equal neighbours at each level (`while (a[i] === a[i-1]) i++`) so you don't emit the same triple twice.

## When to use / when not

- **Hash complement**: unsorted input, you need indices, or you cannot afford the O(n log n) sort. Also the natural choice for "does a pair exist" over a stream.
- **Two-pointer**: input already sorted, memory-constrained, or you're doing 3-/k-sum where sorting is required anyway.

## Interview pitfalls & gotchas

- **Insert-before-check bug**: storing `x` before testing `need` makes an element satisfy `2x === target` against itself. Always check, then insert.
- **Duplicate values**: `Map` keyed by value overwrites the earlier index. Usually fine (you only need *some* valid pair), but if the answer is the *pair of equal values* (e.g. `[3,3]`, target 6), the check-then-insert order still works because the first 3 is stored, and the second 3 finds it.
- **Returning the value vs the index**: two-pointer on a sorted copy loses positions — a frequent off-by-one/index-mismatch source.
- **Overflow**: not an issue with JS `number` for typical ranges, but `target − x` in fixed-width languages can overflow; note it if translating.
- **All pairs vs one pair**: complement hashing finds *one* efficiently; enumerating *all* distinct pairs is better served by the sort + two-pointer sweep with duplicate skipping.

## See Also

- [[hash-maps-and-sets]] — the O(1) membership structure this pattern rides on
- [[hash-tables]] — why lookups are expected O(1)
- [[two-pointer]] — the sorted-array alternative and the 3-sum inner loop
- [[sorting-algorithms]] — the enabling step for k-sum
- [[07-performance-engineering/big-o-in-practice|Big-O in Practice]] — the space-for-time trade in context
