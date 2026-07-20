# Binary Search Trees

A binary search tree (BST) is a [[binary-trees|binary tree]] that maintains an **ordering invariant**, turning a tree into a searchable ordered map/set: it gives you `O(h)` lookup, insert, delete, *and* ordered iteration and range queries — the combination a [[hash-maps-and-sets|hash map]] cannot offer.

## The ordering invariant

For every node `x`: all keys in `x.left` are `< x.key`, and all keys in `x.right` are `> x.key`. Crucially this holds **recursively for entire subtrees**, not just immediate children — that's what lets you discard half the tree at each comparison, the same halving idea as [[binary-search]] but over a linked structure you can mutate cheaply.

An immediate consequence: an **in-order** traversal visits keys in sorted ascending order. This single property is the reason BSTs exist — it makes `min`/`max`, successor/predecessor, k-th smallest, and range scans natural (see [[tree-traversals]]).

## Search and insert

Both walk one root-to-leaf path, comparing and branching:

```ts
function insert(root: Node | null, k: number): Node {
  if (!root) return new Node(k);
  if (k < root.key)      root.left  = insert(root.left, k);
  else if (k > root.key) root.right = insert(root.right, k);
  // k === root.key handled per duplicate policy below
  return root;
}
```

Search is identical minus the mutation. Cost is `Θ(h)`.

## Delete — the one with teeth

Deletion has three cases; the third is where candidates stumble:

1. **Leaf** — just detach it.
2. **One child** — splice the child into the node's slot.
3. **Two children** — you cannot simply remove the node. Replace its key with its **in-order successor** (the minimum of the right subtree) or **in-order predecessor** (the maximum of the left subtree), then delete *that* node — which by construction has at most one child, reducing to case 1 or 2.

```ts
// two-children case: pull up the successor
let succ = node.right!;
while (succ.left) succ = succ.left;   // min of right subtree
node.key = succ.key;
node.right = deleteNode(node.right, succ.key);
```

The successor/predecessor is exactly the neighbor in sorted order, so swapping it in preserves the invariant. Alternating (or randomizing) which side you pull from can reduce lopsidedness in a plain BST.

## Complexity — and the degeneration trap

All three operations are `Θ(h)`. Under balance, `h = Θ(log n)`. But a plain BST does **nothing** to enforce balance:

- Insert **already-sorted** (or reverse-sorted) keys and every node becomes a right (or left) child — the tree degenerates into a [[linked-lists|linked list]] with `h = n − 1`, and every operation is `O(n)`.
- Random insertion order gives expected height `Θ(log n)`, but you rarely control input order, and sorted input is common (timestamps, autoincrement IDs). This is not a rare edge case — it's the default failure in production.

This is precisely why self-balancing variants exist: [[avl-trees]] (strict, `h ≈ 1.44 log₂n`, best for read-heavy loads) and [[red-black-trees]] (looser, fewer rotations per write — the choice behind Java's `TreeMap` and C++'s `std::map`). Reach for those whenever input order is untrusted; treat a hand-rolled unbalanced BST as a teaching tool, not a data structure you ship.

## Duplicate handling — a design decision, not a default

The invariant as stated (`<` / `>`) has no slot for equal keys. Pick a policy explicitly:

- **Forbid** (set semantics) — reject or no-op on equal keys.
- **Count/multiplicity** — store a `count` per node; cheapest and keeps the tree a proper set structurally.
- **Consistent side** — always send equals to (say) the right subtree; simple but skews the tree and complicates deletion. Whatever you choose, comparison and delete logic must agree, or you'll orphan nodes.

## Senior pitfalls

- **Validation**: checking only `left.key < key < right.key` locally is a classic bug — a grandchild can violate the global invariant. Validate by carrying down a `(min, max)` range, or by confirming the in-order traversal is strictly increasing.
- **BST vs hash map**: hashing wins on raw point-lookup average (`O(1)`), but a BST wins when you need **order** — range queries, successor, sorted iteration, `floor`/`ceil`. Don't default to a tree if you never need ordering.
- **Recursion depth** on a degenerate tree overflows the stack — another symptom of skipping balancing.
- **Delete correctness** is the most-failed whiteboard task; rehearse the two-children successor swap until it's reflexive.

## See Also
- [[binary-trees]] — the underlying structure and the `O(h)` vs `O(log n)` distinction
- [[tree-traversals]] — in-order = sorted order; how iteration works
- [[avl-trees]] — strict balancing that kills the degeneration trap
- [[red-black-trees]] — looser balancing favored for write-heavy maps
- [[hash-maps-and-sets]] — the unordered alternative; when to prefer it
- [[binary-search]] — the same halving principle over a sorted array
