# Palindrome Techniques

A palindrome reads the same forwards and backwards. Three techniques cover almost every palindrome problem: **two-pointer verification** (is this a palindrome?), **expand-around-center** (find the longest palindromic substring in O(n²)), and **Manacher's algorithm** (the same in O(n)). Knowing all three lets you pick the simplest tool that meets the constraint.

## Two-pointer check — O(n)

Converge from both ends; the invariant is that everything *outside* `[lo, hi]` has already matched as a mirrored pair.

```ts
function isPalindrome(s: string): boolean {
  let lo = 0, hi = s.length - 1;
  while (lo < hi) {
    if (s[lo] !== s[hi]) return false; // mismatch kills it early
    lo++; hi--;
  }
  return true; // pointers crossed → all pairs matched
}
```

O(n) time, O(1) space. Real variants filter non-alphanumerics or lowercase in place by advancing the pointers past skipped characters.

## Expand-around-center — O(n²)

Every palindrome has a center. A string of length `n` has **2n − 1** centers: `n` single-character centers (odd-length palindromes) and `n − 1` gaps between characters (even-length palindromes). From each center, expand outward while the mirrored characters match.

```
"babad": centers include index 2 ('b') → expand → "aba" (len 3)
```

```ts
function longestPalindrome(s: string): string {
  let start = 0, maxLen = 0;
  const expand = (l: number, r: number) => {
    while (l >= 0 && r < s.length && s[l] === s[r]) { l--; r++; }
    // loop overshoots by one on each side → true span is [l+1, r-1]
    if (r - l - 1 > maxLen) { maxLen = r - l - 1; start = l + 1; }
  };
  for (let i = 0; i < s.length; i++) {
    expand(i, i);     // odd center at i
    expand(i, i + 1); // even center between i and i+1
  }
  return s.slice(start, start + maxLen);
}
```

**Time O(n²)**: 2n − 1 centers, each expansion up to O(n). **Space O(1)** (excluding the output). This beats the O(n²) time / O(n²) space DP table and is the pragmatic interview answer.

## Manacher's algorithm — O(n)

Manacher gets the longest palindromic substring in **linear time** by never re-checking characters an earlier palindrome already proved symmetric.

**Transform** to unify odd/even: interleave a sentinel, e.g. `"aba"` → `"^#a#b#a#$"`. Now every palindrome in the transformed string is odd-length and centered on a single index, and `p[i]` (the radius) directly gives the original length.

**Core trick — mirror reuse.** Track the rightmost palindrome discovered, by its `center` and right edge `right`. For a new index `i` inside that palindrome, its mirror is `mirror = 2*center − i`. Because the big palindrome is symmetric, `p[i]` starts at *at least* `min(right − i, p[mirror])` — those characters are already known to match. You only expand *beyond* that free radius.

```ts
function manacher(s: string): number {
  const t = "^#" + s.split("").join("#") + "#$"; // sentinels avoid bounds checks
  const p = new Array(t.length).fill(0);
  let center = 0, right = 0, best = 0;
  for (let i = 1; i < t.length - 1; i++) {
    if (i < right) {
      const mirror = 2 * center - i;
      p[i] = Math.min(right - i, p[mirror]); // reuse known symmetry
    }
    // expand only past the free radius
    while (t[i + p[i] + 1] === t[i - p[i] - 1]) p[i]++;
    if (i + p[i] > right) { center = i; right = i + p[i]; } // extend rightmost
    best = Math.max(best, p[i]);
  }
  return best; // p[i] in transformed string == longest palindrome length in s
}
```

## Dry run / trace (mirror reuse)

`s = "aaa"` → `t = "^#a#a#a#$"`. Suppose we have expanded around the middle `#`/`a` region so `center=4, right=7`. Reaching `i=5`: `mirror = 2*4 − 5 = 3`, and `p[3]` is already known. We set `p[5] = min(right−i, p[mirror]) = min(2, p[3])` for **free**, then try to expand only beyond that — most characters are never re-compared. That reuse is exactly why the total work is linear.

## Complexity — why Manacher is linear

Each iteration does O(1) bookkeeping plus a `while` loop. The subtle point: **`right` never moves left, and every successful expansion step advances `right`.** Across the whole run the `while` loop can advance `right` at most `n` times total (amortized), so the sum of all expansions is O(n). Hence **O(n) time, O(n) space** (the `p` and `t` arrays). Contrast: expand-around-center re-expands overlapping regions repeatedly → O(n²).

## Variants & trade-offs

- **Longest palindromic *subsequence*** (non-contiguous) is a different beast — it's [[dynamic-programming]], O(n²) via LCS of `s` and `reverse(s)`.
- **Count all palindromic substrings**: expand-around-center counts each successful expansion.
- **Check palindrome under edits** (delete one char, etc.): two-pointer with a branch on mismatch.

## When to use / when not

- Just verifying → **two-pointer**.
- Longest substring, `n` up to a few thousand → **expand-around-center** (simple, O(1) space, hard to get wrong).
- `n` large or O(n) mandated → **Manacher**. It's easy to misimplement, so only reach for it when the constraint truly demands linear time.

## Interview pitfalls & gotchas

- **Odd/even confusion** in expand-around-center: forgetting the `expand(i, i+1)` even-center call misses palindromes like `"abba"`.
- **Off-by-one after expansion**: the loop overshoots by one on each side; the true span is `[l+1, r-1]`, length `r − l − 1`. A classic bug.
- **Manacher sentinels**: the `^`/`$` guards let the inner `while` skip explicit bounds checks — omitting them causes out-of-range reads.
- **Transformed vs original indices**: `p[i]` equals the palindrome length in the *original* string only because of the `#` interleaving; mapping the center back to original coordinates trips people up.
- **Unicode**: `s[i]` indexes UTF-16 code units in JS, so surrogate-pair emoji break naive palindrome checks — use `[...s]` if that matters.

## See Also

- [[two-pointer]] — the convergence pattern behind the O(n) check
- [[string-algorithms]] — sibling string techniques (KMP, Z-function share the "reuse prior matches" idea)
- [[dynamic-programming]] — the DP table variant and palindromic *subsequence*
- [[sliding-window]] — the other main substring-scanning pattern
