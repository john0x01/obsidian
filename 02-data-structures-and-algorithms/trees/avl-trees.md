# AVL Trees

An AVL tree is a self-balancing [[binary-search-trees|binary search tree]] (the first ever invented, Adelson-Velsky & Landis, 1962) that guarantees `O(log n)` search, insert, and delete by keeping the tree **strictly height-balanced** at every node. It exists to eliminate the degeneration trap of a plain BST, where sorted input collapses the tree into an `O(n)` list.

## The balance-factor invariant

For every node, define its **balance factor** as `height(left) − height(right)`. The AVL invariant is:

> `|balanceFactor(node)| ≤ 1` for **every** node.

That is, at no node may the two subtrees' heights differ by more than one. Each node caches its height (or its balance factor) so this is checkable in `O(1)` after an update. This local, per-node constraint is what globally bounds the height.

## Why the height stays logarithmic

The invariant forces the sparsest legal AVL tree to still be fairly full. Let `N(h)` be the *minimum* number of nodes in an AVL tree of height `h`. A minimal tree has one subtree of height `h−1` and one of height `h−2` (the maximum allowed difference), giving the Fibonacci-like recurrence:

```
N(h) = N(h-1) + N(h-2) + 1,   N(0) = 1, N(1) = 2
```

This grows exponentially (`N(h) ≈ φ^h`), so inverting it bounds the height by roughly

> `h ≤ 1.44 · log₂(n + 2) − 0.328  ≈  1.44 · log₂ n`

So an AVL tree is at most ~44% taller than a perfectly balanced tree — a small constant factor, and firmly `O(log n)`. Contrast [[red-black-trees]], whose looser rule allows up to `2·log₂(n+1)`, i.e. roughly twice the perfect height.

## The four rotations

After an insert or delete you walk back up toward the root, updating heights; at the first node whose `|bf|` becomes 2, you rebalance with a rotation. There are four cases, named by the shape of the path to the newly-heavy grandchild:

- **LL** (left-left, left subtree too tall on its left) → single **right** rotation about the unbalanced node.
- **RR** (right-right) → single **left** rotation.
- **LR** (left subtree too tall, but on its right) → **left** rotation on the left child, then **right** rotation on the node.
- **RL** (right subtree too tall on its left) → **right** rotation on the right child, then **left** rotation on the node.

A rotation is a local `O(1)` pointer relink that preserves the [[binary-search-trees|BST]] ordering while reducing height:

```ts
function rotateRight(y: Node): Node {
  const x = y.left!;
  y.left = x.right;      // x's right subtree moves under y
  x.right = y;           // y becomes x's right child
  updateHeight(y);       // order matters: y first (now lower)
  updateHeight(x);
  return x;              // x is the new subtree root
}
```

The mnemonic: LL/RR need one rotation; LR/RL are the "zig-zag" cases needing two. An insert triggers **at most one** rotation (single or double) to restore balance for the whole tree; a delete may trigger rotations at multiple ancestors as you propagate up — up to `O(log n)` of them.

## Complexity

- **Search / insert / delete**: `O(log n)` worst case — guaranteed, not amortized or average, because the height bound is a hard invariant.
- **Rotations per insert**: ≤ 1 (constant). **Per delete**: `O(log n)` in the worst case.
- **Space**: `O(n)`, plus one small integer (height or 2-bit balance factor) per node.
- Building from `n` inserts: `O(n log n)`.

## AVL vs red-black — the real trade-off

Both are `O(log n)`. The distinction is where they spend effort:

- AVL is **more rigidly balanced**, so its height is smaller → **faster lookups**. Prefer AVL for **read-heavy / lookup-dominated** workloads (e.g. an in-memory index queried far more than mutated).
- The strictness costs **more rotations and height maintenance on writes**. [[red-black-trees]] tolerate more imbalance, do fewer rotations per update (`O(1)` amortized recolor-heavy fix-ups), and are therefore the default in write-mixed general-purpose libraries (`std::map`, Java `TreeMap`, Linux scheduler).

So: AVL when you read much more than you write; red-black when writes are frequent or you just want a solid general default.

## Senior pitfalls

- **Update heights in the right order** inside a rotation (the lowered node first), or balance factors go stale and cascade into corruption.
- **Deletion is materially harder than insertion**: one delete can require rebalancing at every ancestor, and it's easy to stop after the first rotation. Test with adversarial sequences, not just inserts.
- Storing full heights vs a 2-bit balance factor is a real memory trade-off at scale; the 2-bit form is trickier to maintain.
- Rotations wreck **cache locality** and pointer-chase (see [[07-performance-engineering/data-locality|Data Locality]]) — for on-disk or very large in-memory ordered data, a B-tree's high fan-out usually beats any binary balanced tree.

## See Also
- [[binary-search-trees]] — the base structure and the degeneration problem AVL solves
- [[red-black-trees]] — the looser-balance alternative; when to prefer it
- [[binary-trees]] — height vs `n` and why balance matters
- [[b-trees-and-b-plus-trees]] — high-fan-out balancing for disk/large data
