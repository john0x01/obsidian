# Space Complexity

Space complexity measures how much memory an algorithm consumes as a function of input size, and it is the axis engineers most often forget until an out-of-memory kill or a stack overflow reminds them. It solves the same modeling problem as time complexity — machine-independent scalability reasoning — but for the memory dimension, where the constants (bytes per node, allocator overhead) and the failure modes (stack limits, GC pressure) are entirely different.

## Auxiliary vs Total Space

Two distinct quantities, and quoting the wrong one is a classic error:

- **Total space** = input + auxiliary + output.
- **Auxiliary space** = extra memory the algorithm allocates beyond the input.

When we say mergesort is "`O(n)` space" we mean *auxiliary* — it needs an `O(n)` scratch buffer to merge. Its total space is also `O(n)` because the input is `Θ(n)`. The distinction matters most for algorithms whose input is huge but whose working set is tiny: a streaming maximum over `n` numbers is `O(1)` auxiliary though it reads `Θ(n)` input. Always state which you mean; "space complexity" unqualified almost always intends auxiliary.

## Recursion and Call-Stack Space

Recursion consumes memory invisibly: every live call frame occupies the stack until it returns. **Stack space = maximum recursion depth × frame size.** This is the hidden `O(depth)` term people omit when they call a recursive algorithm "in-place."

- Recursive [[binary-search]]: `O(log n)` stack from `log n` nested calls (even though it allocates nothing on the heap).
- [[recursion|Recursive]] [[sorting-algorithms|quicksort]]: `O(log n)` stack if you recurse into the *smaller* partition and loop on the larger; `O(n)` worst case if you naïvely recurse into both and hit the degenerate pivot.
- [[tree-traversals|Tree traversal]]: `O(h)` stack where `h` is height — `O(log n)` balanced, `O(n)` for a degenerate/skewed tree.

```ts
// O(log n) STACK space even though nothing is heap-allocated.
function bsearch(a: number[], t: number, lo = 0, hi = a.length - 1): number {
  if (lo > hi) return -1;
  const mid = lo + ((hi - lo) >> 1);          // avoids (lo+hi) overflow
  if (a[mid] === t) return mid;
  return a[mid] < t ? bsearch(a, t, mid + 1, hi)
                    : bsearch(a, t, lo, mid - 1); // deepest chain = the stack
}
```

Deep recursion overflows the stack far sooner than the heap fills — on a typical runtime, a chain of `~10⁴–10⁵` frames. Converting to iteration with an explicit stack moves that memory to the heap (where limits are far higher and failures are catchable) and enables the compiler-free languages' equivalent of tail-call elimination — see [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]].

## "In-Place" — Definition and Caveats

An algorithm is **in-place** if it uses `O(1)` auxiliary space (some texts relax this to `O(log n)`). The caveats trip up seniors constantly:

- **Recursion breaks the `O(1)` claim.** In-place quicksort still uses `O(log n)` stack. "In-place" usually means "no auxiliary *heap* array," silently excusing the stack.
- **Rebuilding vs mutating.** In a language with immutable strings (JS/TS strings are immutable), an "in-place string reverse" is a lie — every mutation allocates a new string. `O(1)`-space claims assume a mutable buffer.
- **Hidden closures/iterators.** Generators, closures capturing the array, and slice views can retain memory you thought was freed.

## Space–Time Trade-offs

Memory and time are exchangeable, and the exchange rate is a core design lever:

- **[[07-performance-engineering/memoization|Memoization]] / [[dynamic-programming|DP]] tables** buy time with space: turning exponential recomputation into polynomial time by storing subproblem results. Naive Fibonacci is `O(φⁿ)` time / `O(n)` stack; memoized is `O(n)` time / `O(n)` space; and the classic *space optimization* keeps only the last two values → `O(n)` time / **`O(1)`** space. Many DP problems admit this "rolling array" reduction from `O(n·m)` to `O(min(n,m))` because each row depends only on the previous.
- **Precomputation** ([[prefix-sums|prefix sums]], lookup tables, [[hash-tables|hash indexes]]) pays memory up front for `O(1)` queries later.
- **The reverse trade** (recompute instead of store) matters when memory is the binding constraint — e.g. gradient checkpointing, or recomputing a value rather than caching it under GC pressure.

## Output-Sensitive Space

Some algorithms' space is dominated not by input or scratch but by *how big the answer is*. Generating all subsets is `Θ(2ⁿ)` output; listing all paths, all permutations (`Θ(n!)`), or all matches in a search can dwarf the input. Here the right move is often to **stream** results (yield one at a time via a generator) so peak auxiliary space stays `O(depth)` instead of materializing the whole `Θ(output)` collection — [[backtracking]] is the canonical case, keeping only the current candidate `O(n)` on the path rather than all solutions in memory.

## Senior Pitfalls

- Quoting time complexity and forgetting the `O(depth)` recursion stack entirely.
- Claiming "in-place" while allocating a copy in an immutable-data language.
- Ignoring allocator/object-header overhead: a linked list of `n` numbers can cost several times the bytes of an array of `n` numbers, worsening cache behavior too — see [[07-performance-engineering/data-locality|Data Locality]].
- GC languages: allocations you free are still *pressure* until collected; peak resident space, not net, is what triggers the OOM killer.

## See Also

- [[recursion]]
- [[07-performance-engineering/memoization|Memoization]]
- [[dynamic-programming]]
- [[asymptotic-notation]]
- [[recurrence-relations]]
- [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]]
