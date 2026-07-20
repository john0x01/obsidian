# Recurrence Relations & the Master Theorem

A recurrence relation expresses an algorithm's running time in terms of its running time on smaller inputs. It is the natural cost model for [[divide-and-conquer]] and [[recursion|recursive]] algorithms, and the Master Theorem is a cookbook that solves the most common shape in one step — so you can read a `Θ` bound straight off the recursion structure.

## Setting Up a Recurrence

For a divide-and-conquer algorithm that splits an input of size `n` into `a` subproblems each of size `n/b`, does `f(n)` non-recursive work to divide and combine, and stops at a constant base case:

```
T(n) = a · T(n/b) + f(n),   T(1) = Θ(1)
```

Identify the three parameters directly from the code: `a` = number of recursive calls, `b` = shrink factor per call, `f(n)` = everything outside the recursion (partitioning, merging, the loop that scans the input). Mergesort splits into `a = 2` halves (`b = 2`) and merges in linear time (`f(n) = Θ(n)`): `T(n) = 2T(n/2) + Θ(n)`.

## Recursion-Tree Method

Expand the recurrence into a tree: the root does `f(n)` work, its `a` children each do `f(n/b)`, and so on down `log_b n` levels until leaves of size 1. Sum the work *per level*, then sum the levels. This makes the Master Theorem's three cases *visible*: the answer is dominated by the root, spread evenly across all levels, or dominated by the leaves.

```
level 0:                 f(n)                         → a⁰·f(n)
level 1:        f(n/b)  f(n/b) ... (a of them)        → a·f(n/b)
...
level log_b n:  Θ(1) Θ(1) ... (a^{log_b n} leaves)    → n^{log_b a}·Θ(1)
```

There are `a^{log_b n} = n^{log_b a}` leaves. The exponent **`log_b a`** — call it the *watershed* — is what you compare `f(n)` against.

## The Master Theorem

Compare `f(n)` to `n^{log_b a}`:

- **Case 1 (leaf-dominated).** If `f(n) = O(n^{log_b a − ε})` for some `ε > 0` (f is polynomially *smaller*), then `T(n) = Θ(n^{log_b a})`. The leaves dominate.
- **Case 2 (balanced).** If `f(n) = Θ(n^{log_b a} · log^k n)` for some `k ≥ 0`, then `T(n) = Θ(n^{log_b a} · log^{k+1} n)`. Work is even across levels; you gain one log factor from the `log_b n` levels. (The common form is `k = 0`: `f = Θ(n^{log_b a})` → `Θ(n^{log_b a} log n)`.)
- **Case 3 (root-dominated).** If `f(n) = Ω(n^{log_b a + ε})` for some `ε > 0` *and* the **regularity condition** `a · f(n/b) ≤ c · f(n)` holds for some constant `c < 1` and all sufficiently large `n`, then `T(n) = Θ(f(n))`. The root dominates.

The regularity condition in Case 3 is not optional. It guarantees the per-level work actually *shrinks* geometrically toward the leaves; without it a pathological `f` could grow fast enough to fail the theorem even while looking root-heavy. For any polynomially-bounded `f` it holds automatically, which is why it is often glossed over.

## Worked Examples

- **Mergesort:** `T(n) = 2T(n/2) + Θ(n)`. Here `log_b a = log₂ 2 = 1`, and `f(n) = Θ(n) = Θ(n¹)`. Case 2 with `k = 0` → **`Θ(n log n)`**. See [[sorting-algorithms]].
- **Binary search:** `T(n) = T(n/2) + Θ(1)`. `a = 1, b = 2`, so `log_b a = 0` and `n^0 = 1`; `f(n) = Θ(1) = Θ(n^0)`. Case 2 → **`Θ(log n)`**. See [[binary-search]].
- **Strassen's matrix multiply:** `T(n) = 7T(n/2) + Θ(n²)`. `log₂ 7 ≈ 2.807`, and `f(n) = n²` is polynomially smaller. Case 1 → **`Θ(n^{log₂ 7}) ≈ Θ(n^{2.807})`**, beating naive `Θ(n³)`.
- **Naive matrix multiply (recursive, 8 subproblems):** `T(n) = 8T(n/2) + Θ(n²)`, `log₂ 8 = 3`, Case 1 → `Θ(n³)` — the exact win Strassen buys by dropping one multiplication.

## When the Master Theorem Does NOT Apply

- **The gap cases.** If `f(n)` sits *between* polynomial classes — e.g. `f(n) = n/log n` vs `n^{log_b a} = n` — it is neither polynomially smaller nor larger, so none of the three cases fits. (`T(n) = 2T(n/2) + n/log n` solves to `Θ(n log log n)` via the recursion tree, not the Master Theorem.)
- **Unequal subproblem sizes.** `T(n) = T(n/3) + T(2n/3) + n` (a common quickselect/median-of-medians shape) violates the "all subproblems size `n/b`" assumption. Use the **Akra–Bazzi** method, which generalizes the theorem to `Σ aᵢ T(n/bᵢ) + g(n)` by finding the exponent `p` with `Σ aᵢ b_i^{−p} = 1`.
- **Non-constant `a` or `b`, or subtractive recurrences** like `T(n) = T(n−1) + Θ(n)` (→ `Θ(n²)`): solve these directly by unrolling.

## See Also

- [[divide-and-conquer]]
- [[sorting-algorithms]]
- [[binary-search]]
- [[asymptotic-notation]]
- [[recursion]]
- [[space-complexity]]
