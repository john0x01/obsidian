# String Matching Algorithms

String matching asks: where does pattern `P` (length `m`) occur inside text `T` (length `n`)? The naïve approach is O(nm); the classic algorithms — KMP, Rabin–Karp, Z, Boyer–Moore — all attack the same waste: after a partial match fails, the naïve method throws away everything it just learned and restarts one character over. Each algorithm is a different way to *reuse* that information.

## Naïve Matching and Its Waste

Try `P` at every alignment `i` in `0..n-m`, comparing character by character:

```ts
function naive(T: string, P: string): number[] {
  const hits: number[] = [];
  for (let i = 0; i + P.length <= T.length; i++) {
    let j = 0;
    while (j < P.length && T[i + j] === P[j]) j++;
    if (j === P.length) hits.push(i);
  }
  return hits;
}
```

Worst case O(nm) — e.g. `T = "aaaa…a"`, `P = "aaab"`: every alignment matches `m-1` chars then fails. On random text it's near O(n) on average, which is why it survives in practice for short patterns.

## KMP — Reusing the Prefix (Failure) Function

Knuth–Morris–Pratt precomputes, for the *pattern alone*, a **prefix function** `π[j]` = the length of the longest proper prefix of `P[0..j]` that is also a suffix of it. When a mismatch happens at pattern index `j`, instead of restarting, KMP shifts the pattern so its longest matching prefix realigns — i.e. it jumps `j` back to `π[j-1]` without touching the text pointer.

The invariant that makes it linear: **the text index `i` never moves backward.** Each character of `T` is examined a bounded number of times; the total back-jumps in `j` are amortized against forward progress (an aggregate argument like [[amortized-analysis]]). Building `π` is O(m), the scan is O(n), so KMP is **O(n + m) worst case, O(m) extra space** — the failure function is what buys guaranteed linearity that naïve matching lacks. The prefix function itself is independently useful (period detection, longest palindromic prefix).

## Rabin–Karp — Rolling Hash

Rabin–Karp hashes `P` and each length-`m` window of `T`, comparing hashes instead of characters. The win is the **rolling hash**: sliding the window one position updates the hash in O(1) by removing the leaving character's contribution and adding the entering one (polynomial hash mod a large prime):

```ts
// conceptual roll: h_new = (h_old - T[i]*base^(m-1)) * base + T[i+m]  (mod prime)
```

- **Expected O(n + m)**: hash comparison is O(1); a hash *match* triggers an O(m) verification, but with a good hash and large prime, spurious matches are rare.
- **Worst case O(nm)**: adversarial input causing every window to hash-collide forces verification everywhere. **You must verify every hash hit character-by-character** — hash equality is necessary, not sufficient (a *spurious hit* / false positive). Skipping verification is a correctness bug.

Rabin–Karp's real edge is **multi-pattern search** (hash a set of patterns, one text pass) and **2-D / plagiarism-style** matching, where KMP doesn't generalize as cleanly. See [[hash-functions]] for polynomial hashing and collision behavior.

## Z-Algorithm and Boyer–Moore (Briefly)

- **Z-algorithm.** Computes `Z[i]` = length of the longest substring starting at `i` that matches a prefix of the string. Run it on `P + separator + T` and any `Z[i] ≥ m` marks a match. O(n + m), often considered more intuitive than KMP; both encode "longest prefix that's also here" from opposite framings.
- **Boyer–Moore.** Scans the pattern **right-to-left** and uses two heuristics — the *bad-character* rule (jump past a text char absent from the pattern) and the *good-suffix* rule — to **skip** large chunks of text. Best/typical case is **sublinear** (fewer than `n` comparisons — it can skip up to `m` chars per mismatch), which is why `grep` and many editors use Boyer–Moore or its Boyer–Moore–Horspool simplification. Worst case is O(nm) for the basic form (O(n) with Galil's rule). It shines when the alphabet is large and the pattern is long.

## When to Use Which

- Single pattern, guaranteed linear, small alphabet → **KMP** (or Z).
- Long pattern, large alphabet, practical speed → **Boyer–Moore**.
- Many patterns at once, or rolling/2-D/fingerprinting → **Rabin–Karp**.
- Prefix queries, autocomplete, many patterns sharing prefixes → build a [[tries|trie]] (or Aho–Corasick for multi-pattern, KMP generalized to a trie).

## Senior Pitfalls

- **Skipping Rabin–Karp verification** — hash equality ≠ string equality; unverified matches are wrong, and a weak modulus invites adversarial O(nm) collisions (a hash-flooding DoS vector).
- **Off-by-one in the prefix function** — `π` uses *proper* prefixes (excluding the whole string); including the full string yields a degenerate table.
- **Assuming KMP beats naïve in practice** — for short patterns on random text, naïve and `String.indexOf` (often a tuned two-way or Boyer–Moore–Horspool) usually win; KMP's value is the worst-case guarantee.
- **Unicode/encoding** — matching on UTF-16 code units vs code points vs grapheme clusters silently mismatches; normalize first.

## See Also

- [[tries]]
- [[hash-functions]]
- [[hash-tables]]
- [[two-pointer]]
- [[04-databases/search-engines|Search Engines]]
