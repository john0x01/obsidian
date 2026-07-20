# Red-Black Trees

A red-black tree is a self-balancing [[binary-search-trees|binary search tree]] that guarantees `O(log n)` search, insert, and delete by coloring each node red or black and enforcing five color rules. It trades the strict height of an [[avl-trees|AVL tree]] for **fewer structural changes per update**, which is why it's the ordered-map workhorse inside Java's `TreeMap`, C++'s `std::map`/`std::set`, and the Linux Completely Fair Scheduler.

## The five properties

1. Every node is **red or black**.
2. The **root is black**.
3. Every **leaf** (the conceptual `NIL` sentinel) is **black**.
4. A **red node's children are both black** — i.e. no two reds in a row on any path.
5. Every root-to-`NIL` path passes through the **same number of black nodes**.

Properties 4 and 5 together are the whole trick, and it helps to keep the `NIL` leaves as a single shared black sentinel — it simplifies both the invariants and the fix-up code.

## Black-height and the height bound

The **black-height** `bh(x)` is the number of black nodes on any path from `x` down to a `NIL`, not counting `x`. Property 5 makes this well-defined. From here the log bound falls out:

- A subtree rooted at `x` contains at least `2^(bh(x)) − 1` internal nodes (provable by induction).
- By property 4, no path can be more than twice as long as any other from the same node (reds can't stack), so `bh(root) ≥ h/2`.
- Combining: `n ≥ 2^(h/2) − 1`, which rearranges to

> `h ≤ 2 · log₂(n + 1)`.

So the height is at most **twice** the perfect-tree height — looser than AVL's `≈ 1.44 log₂n`, but still `O(log n)`. That extra slack is deliberate: it's what lets updates get away with less work.

## Insert and delete fix-up — the intuition

Every insert starts as a standard BST insert, and the new node is colored **red** (adding a red node never breaks property 5, only possibly property 4). If its parent is black, you're done. If the parent is also red, you have a "red-red violation" and fix it by looking at the **uncle** (the parent's sibling):

- **Uncle is red** → **recolor**: flip parent and uncle to black, grandparent to red, then recurse upward from the grandparent. This is cheap and can cascade up the tree, but does **no rotations**.
- **Uncle is black** (or `NIL`) → **rotate**: a single or double rotation around the grandparent (the LL/LR/RL/RR shapes, same as [[avl-trees]]), plus a recolor, and the violation is resolved locally — no further propagation.

The key outcome: an insert needs **at most 2 rotations** and `O(log n)` recolorings, but the recolorings are `O(1)` pointer-free flips. **Delete** is genuinely intricate — after splicing out a node you may create a "doubly-black" deficit that violates property 5, resolved through a handful of sibling-color cases (mirror-imaged left/right) with recolors and **at most 3 rotations**. The cases are why you describe delete rather than write it from memory; the takeaway is that rotations stay bounded by a small constant.

## Complexity

- **Search / insert / delete**: `O(log n)` worst case (hard bound via the height guarantee, not amortized).
- **Rotations per insert**: ≤ 2. **Per delete**: ≤ 3. Recolorings: `O(log n)` but cheap.
- **Space**: `O(n)` + **1 bit** of color per node (often stolen from a pointer's low bit — no extra word), versus AVL's height/balance field.

## Red-black vs AVL — why libraries pick red-black

Both are `O(log n)`; the difference is where the cost lands:

- AVL is **shorter** (`1.44 log₂n` vs `2 log₂n`) → wins on **read-heavy** workloads because lookups touch fewer nodes.
- Red-black does **fewer rotations on writes** and leans on cheap recoloring, so it wins on **insert/delete-heavy or mixed** workloads.

General-purpose standard libraries face unknown, write-mixed usage, so they choose the structure that never pays too much on any single operation — red-black. That's the rationale behind `std::map`, `TreeMap`, and the CFS run-queue keyed on task virtual runtime.

For **on-disk or very large** ordered data the answer is neither: a high-fan-out [[04-databases/b-trees|B-Tree (Databases)]] (or B+ tree) minimizes block/page reads, which dominates over comparison count once data doesn't fit in cache — see also [[04-databases/indexing|Indexing]].

## Senior pitfalls

- **Delete fix-up is the hard part** — the doubly-black cases are famously bug-prone and mirror-symmetric; lean on a tested library rather than a hand-roll in production.
- Treating **`NIL` as a real shared black sentinel** avoids a swarm of null checks and makes property 3/5 reasoning uniform.
- The height bound is `2 log₂(n+1)`, **not** balanced-tree-optimal — don't quote AVL's constant for a red-black tree.
- Like all pointer-based balanced trees, rotations thrash **cache locality** ([[07-performance-engineering/data-locality|Data Locality]]); Big-O hides this constant, and it's why B-trees win on disk.

## See Also
- [[binary-search-trees]] — the base structure and the balancing motivation
- [[avl-trees]] — the stricter alternative; read-heavy vs write-heavy trade-off
- [[binary-trees]] — height vs `n` fundamentals
- [[04-databases/b-trees|B-Trees (Databases)]] — high-fan-out balancing for disk-resident indexes
- [[04-databases/indexing|Indexing]] — where ordered structures power range queries
