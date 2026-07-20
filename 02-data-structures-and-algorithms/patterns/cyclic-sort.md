# Cyclic Sort

Cyclic sort is a niche but powerful pattern for arrays that hold a **permutation of `1..n`** (or `0..n−1`) — possibly with a missing or duplicate value. It sorts by placing each element at the index equal to its value, using only swaps, in **O(n) time and O(1) extra space**. Its real payoff isn't sorting: it's the "index-as-hash" trick that solves find-missing / find-duplicate / first-missing-positive without any auxiliary structure.

## The idea / invariant

If the values are exactly `1..n`, then value `v` belongs at index `v − 1` (or index `v` for `0..n−1`). Walk the array; whenever `a[i]` is not already home, **swap it to its correct index**. Don't advance `i` after a swap — you just pulled a new, possibly-misplaced value into slot `i`, so re-examine it. Advance only when `a[i]` is already in place.

**Key invariant:** *every swap sends at least one value to its final, correct position.* A value never leaves a correct slot once placed. Since a position, once fixed, is never disturbed, there can be at most **n − 1 swaps** total — that's what makes the whole thing linear despite the nested-looking structure.

```
target: value v → index v-1
[3, 1, 2]  i=0: a[0]=3 → wants index 2, swap
[2, 1, 3]  i=0: a[0]=2 → wants index 1, swap
[1, 2, 3]  i=0: a[0]=1 → home, advance i
```

## Implementation

```ts
function cyclicSort(a: number[]): void {
  let i = 0;
  while (i < a.length) {
    const correct = a[i] - 1;        // value v belongs at index v-1
    if (a[i] !== a[correct]) {       // not home AND not a duplicate stalemate
      [a[i], a[correct]] = [a[correct], a[i]]; // swap into place, don't advance
    } else {
      i++;                           // a[i] correct (or a duplicate) → move on
    }
  }
}
```

The guard `a[i] !== a[correct]` (comparing *values*, not `i !== correct`) is what makes the loop terminate on duplicates: if the target slot already holds the same value, swapping would loop forever, so we advance instead.

## Dry run / trace

`cyclicSort([3, 1, 2])`, all 1-indexed values, `correct = a[i] − 1`:

| i | array | `a[i]` | `correct` | `a[correct]` | action |
|---|-------|--------|-----------|--------------|--------|
| 0 | `[3,1,2]` | 3 | 2 | 2 | `3 ≠ 2` → swap i↔2 |
| 0 | `[2,1,3]` | 2 | 1 | 1 | `2 ≠ 1` → swap i↔1 |
| 0 | `[1,2,3]` | 1 | 0 | 1 | `1 == 1` → advance |
| 1 | `[1,2,3]` | 2 | 1 | 2 | home → advance |
| 2 | `[1,2,3]` | 3 | 2 | 3 | home → advance |

Sorted in 2 swaps (≤ n − 1 = 2). Notice `i` stayed at 0 through both swaps — that's the "re-examine after swap" behaviour.

## Complexity — why O(n) with O(1) space

The `while` loop looks like it could be quadratic, but count *work*: each iteration either does a swap or increments `i`. Increments happen at most `n` times. Swaps happen at most `n − 1` times (each fixes a position permanently). So total iterations ≤ `2n − 1` = **O(n) time**. Only a constant number of index variables are used → **O(1) extra space** (in place). It is **not stable** and only defined for the restricted value domain.

## The index-as-hash idea — what it unlocks

Because values map bijectively to indices, the array *is* its own hash table. After cyclic sort, scan once: any position `i` where `a[i] !== i + 1` reveals an anomaly. This solves a family of problems in O(n)/O(1):

- **Find the missing number** (`1..n` with one gone): after sorting, the first index where `a[i] !== i + 1` is missing value `i + 1`.
- **Find the duplicate**: the value sitting where its rightful owner should be — during the swap phase, when `a[i] !== i` but the target already holds an equal value, `a[i]` is the duplicate.
- **First missing positive** (unsorted, unbounded, negatives present): place every value in `1..n` at its home index (ignore out-of-range and non-positive values), then the first index `i` with `a[i] !== i + 1` gives answer `i + 1`. This is the canonical "beat the O(n) extra-space set" question.
- **Find all missing / all duplicates** in `1..n`: one cyclic pass, then collect the mismatches.

```ts
function findMissing(a: number[]): number { // a is a permutation of 1..n missing one, length n
  cyclicSort(a); // note: needs a domain-safe variant that skips out-of-range values
  for (let i = 0; i < a.length; i++) if (a[i] !== i + 1) return i + 1;
  return a.length + 1;
}
```

(For the "first missing positive" variant, guard the swap with `a[i] > 0 && a[i] <= n` so out-of-range junk is left alone.)

## Variants & trade-offs

- **0-indexed domain** (`0..n−1`): `correct = a[i]` instead of `a[i] − 1`.
- **Out-of-range tolerant**: add `a[i] > 0 && a[i] <= n && …` to the swap guard — essential for first-missing-positive.
- **Alternatives**: the same problems yield to a `Set` (O(n) time, O(n) space) or, for find-the-duplicate, [[fast-slow-pointers|Floyd's cycle detection]] treating values as `next` pointers (O(n)/O(1) but read-only). Cyclic sort's edge is O(1) space *plus* enumerating every anomaly.

## When to use / when not

Use it precisely when the input is a (near-)permutation of a contiguous integer range and you're asked for missing/duplicate/first-positive under tight space. **Don't** use it when values aren't a bounded contiguous range, when the array must stay immutable, or when values map to indices ambiguously — the whole trick collapses without the value↔index bijection.

## Interview pitfalls & gotchas

- **Advancing `i` after a swap**: the top bug. You must *re-process* index `i` after swapping, because a fresh unplaced value now sits there. Only `i++` when the slot is correct.
- **Guarding on indices vs values**: `if (i !== correct)` loops forever on duplicates; compare **values** (`a[i] !== a[correct]`) so equal-value stalemates advance instead.
- **Off-by-one on the domain**: `1..n` uses `a[i] − 1`; `0..n−1` uses `a[i]`. Mixing them corrupts every placement.
- **Out-of-range access**: for first-missing-positive, unguarded `a[a[i] − 1]` on a value like `10^9` throws or misplaces — always range-check before swapping.
- **Assuming a total sort**: cyclic sort only orders a permutation domain; it's not a general-purpose sort.

## See Also

- [[two-pointer]] — sibling in-place, O(1)-space array technique
- [[hash-maps-and-sets]] — the O(n)-space alternative cyclic sort replaces
- [[in-place-reversal]] — another swap-based, O(1)-space array manipulation
- [[fast-slow-pointers]] — Floyd's read-only alternative for find-the-duplicate
- [[sorting-algorithms]] — where cyclic sort sits among the O(n) non-comparison sorts
