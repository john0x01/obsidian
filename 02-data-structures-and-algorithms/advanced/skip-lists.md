# Skip Lists

A skip list is an ordered dictionary — search, insert, delete, and range/predecessor queries — built from stacked linked lists rather than a balanced tree. It solves the same problem as a balanced [[binary-search-trees|BST]] (keep sorted data with logarithmic operations) but reaches balance through **randomization** instead of rotations, which makes it dramatically simpler to reason about and to make concurrent.

## The Idea: Express Lanes Over a Sorted List

Start with an ordinary sorted [[linked-lists|linked list]] — that gives O(n) search because you must walk node by node. Now add *express lanes*: a sparse list above it that skips over many nodes, then a sparser lane above that, and so on. Searching descends this layered structure like an interstate: ride the top lane until the next node would overshoot your key, drop down one level, repeat, until you land at the bottom level.

```
L3: HEAD --------------------------------> 25 ------------> NIL
L2: HEAD ---------> 9 -------------------> 25 ------> 40 --> NIL
L1: HEAD --> 3 --> 9 ------> 17 --------> 25 ------> 40 --> NIL
L0: HEAD --> 3 --> 9 --> 12 --> 17 --> 21 --> 25 --> 33 --> 40 --> NIL   (full sorted list)
```

The trick is **how tall each node is**. On insertion, a node always joins level 0; then you flip a coin (probability *p*, usually 1/2). Heads → promote it one level up and flip again; tails → stop. So a node reaches level *i* with probability *pⁱ*. No global rebalancing, no rotations, no color bits — a node's height is decided locally at insertion time by coin flips and never changes.

```ts
function randomLevel(): number {
  let lvl = 0;
  while (Math.random() < P && lvl < MAX_LEVEL) lvl++;   // geometric: E[height]=1/(1-P)
  return lvl;
}
// search: from top level, advance while next.key < target, else drop down a level
```

**The invariant** is purely structural, not a balance condition: every level is a sorted sublist, and level *i+1*'s keys are a subset of level *i*'s. Correctness never depends on the coin flips — only *performance* does.

## Complexity — Expected, Not Worst-Case

- **Search / insert / delete: O(log n) *expected*, O(n) worst case.**

State the assumption precisely: the log bound is an **expectation taken over the random coin flips**, not over the input distribution and *not* a worst-case guarantee. With *p* = 1/2 there are expected log₂ n levels, and at each level you take an expected constant number (≈ 1/p) of forward hops before dropping down — giving Θ(log n) expected total work. The genuinely bad case (every node flips to height 0, degenerating into one flat list, or all to max height) is astronomically unlikely — its probability decays exponentially in *n* — so unlike quicksort there is **no adversarial input** that forces the worst case; only unlucky randomness can, and the coins are internal. This is the same flavor of guarantee as randomized quicksort and treaps.

- **Space: O(n) expected**, but with real overhead: expected total pointers = n/(1−p), so ≈ 2n forward pointers at p=1/2 — more than a BST's ~2n child pointers only modestly, tunable down by lowering *p* (fewer levels, taller search).

## Variants & Trade-offs vs Balanced Trees

- **vs [[red-black-trees]] / [[avl-trees]]:** trees give *worst-case* O(log n); skip lists give *expected* O(log n). Trees win on hard guarantees; skip lists win on **implementation simplicity** — no rotation cases to get wrong — and, crucially, on **concurrency**.
- **Concurrency is the real selling point.** A tree rotation restructures a whole subtree, so lock-free tree updates are notoriously hard. A skip-list insert/delete only relinks a few `next` pointers at each level — localized, single-pointer CAS-friendly updates — which makes lock-free and fine-grained-locking implementations far more tractable. This is why the concurrent ordered map in the JDK, **`ConcurrentSkipListMap`**, is a skip list rather than a tree, and why **Redis sorted sets (ZSET)** use a skip list (plus a hash) for their ordered index.
- **Deterministic skip lists** exist (fixing gaps between promoted nodes) to remove the randomness, but they lose the simplicity that motivated the structure.
- **p is a dial:** smaller *p* → less space and fewer levels but more hops per level; p=1/2 or 1/4 are typical.

## When To Use It & Pitfalls

Reach for a skip list when you need an ordered map with range scans **and** high concurrency, or simply want a balanced-tree alternative that's easy to implement correctly. Prefer a balanced tree when you need worst-case latency guarantees, and a plain [[hash-tables|hash table]] when you don't need ordering at all.

- **"Expected" is not "always."** For hard-real-time or adversarial latency SLAs, the tail is unbounded in principle — use a tree.
- **RNG quality matters:** a low-entropy or predictable generator can skew heights and degrade performance; and a *seedable, attacker-known* RNG reintroduces a (mild) adversarial angle for level distribution.
- **Cache locality is worse than a B-tree's:** like any pointer-chasing structure it scatters across the heap, so on-disk or cache-sensitive workloads still favor [[b-trees-and-b-plus-trees|B-trees]] (see [[big-o-vs-reality]]).

## See Also
- [[linked-lists]] — the base layer a skip list stacks upon
- [[binary-search-trees]] — the ordered-dictionary problem it also solves
- [[red-black-trees]] — worst-case-balanced alternative it competes with
- [[hash-tables]] — the unordered O(1) alternative when order isn't needed
- [[big-o-vs-reality]] — why pointer-chasing loses to B-trees on real hardware
