# Binary Trees

A binary tree is a hierarchical structure in which every node has at most two children, conventionally named `left` and `right`. It is the substrate for ordered lookups ([[binary-search-trees]]), priority queues ([[heaps]]), prefix indexes ([[tries]]), and expression evaluation — anything whose shape is naturally recursive and where you want to prune half the work at each step.

## Terminology (get these exact)

- **Root** — the single node with no parent. **Leaf** — a node with no children. **Internal** node — one with at least one child.
- **Depth** of a node = number of edges from the root to it (root has depth 0). **Height** of a node = number of edges on the longest downward path to a leaf (a leaf has height 0). **Height of the tree** = height of the root. An empty tree is usually defined as height −1 so recurrences work out.
- **Size** = node count `n`. **Edges** = `n − 1` (a tree is a connected acyclic graph — see [[graphs]]).

The single most important quantity is **height `h`**, because almost every operation is bounded by it.

## The shape zoo — and why the labels matter

- **Full** (a.k.a. proper/strict): every node has 0 or 2 children — never exactly one.
- **Complete**: every level is full except possibly the last, which is filled left-to-right. This is the shape that makes the array representation dense and gapless — it's exactly the [[heaps]] invariant.
- **Perfect**: every internal node has two children and all leaves share the same depth. A perfect tree of height `h` has exactly `2^(h+1) − 1` nodes, so `h = log₂(n+1) − 1`.
- **Balanced**: height is `O(log n)` — the children's heights differ by a bounded amount at every node (the precise bound is what [[avl-trees]] and [[red-black-trees]] enforce).
- **Degenerate** (pathological): each node has one child, so the tree is effectively a [[linked-lists|linked list]] with `h = n − 1`.

"Balanced" vs "degenerate" is the whole ballgame for complexity — see below.

## Representations

**Linked nodes** — the general form. Flexible for sparse or arbitrarily-shaped trees, but each node is a separate heap allocation, so traversal chases pointers and suffers cache misses (see [[07-performance-engineering/data-locality|Data Locality]]).

```ts
class TreeNode<T> {
  val: T;
  left: TreeNode<T> | null = null;
  right: TreeNode<T> | null = null;
  constructor(v: T) { this.val = v; }
}
```

**Array (implicit) representation** — store the tree level-by-level in an array. For a node at index `i`:

```ts
const left  = 2 * i + 1;   // left child
const right = 2 * i + 2;   // right child
const parent = (i - 1) >> 1; // floor((i-1)/2)
```

No pointers, perfect locality, and arithmetic navigation — but it only stays compact for a **complete** tree. A degenerate tree in an array wastes `O(2^n)` slots, which is why this layout is reserved for heaps, not general trees.

## Why operations are O(h)

Search, insert, and delete on a search-structured binary tree walk a single root-to-node path: at each step you descend to one child, doing `O(1)` work. The path length is at most the height, so the cost is `Θ(h)`. That is the exact bound — no averaging, no hidden constants beyond the per-node comparison.

The catch is the relationship between `h` and `n`:

- **Balanced**: `h = Θ(log n)`, so operations are `O(log n)`. A perfect tree gives the floor; any tree kept within a constant factor of it (AVL, red-black) keeps the log.
- **Degenerate**: `h = n − 1`, so the same operations are `O(n)` — no better than scanning a list.

So "`O(log n)` binary-tree operations" is a **conditional** claim that holds *only* under a balance invariant that something actively maintains. A plain BST fed sorted input degenerates silently; this is precisely the failure mode that motivates self-balancing trees. Don't quote `O(log n)` for a structure that has no rebalancing.

## Senior pitfalls

- **Recursion depth**: traversals recurse to depth `h`. On a degenerate tree that's `O(n)` stack frames — a real stack-overflow risk on large adversarial inputs. Prefer an explicit [[stacks|stack]] or Morris traversal when `h` isn't bounded (see [[tree-traversals]]).
- **Height vs depth confusion** flips base cases; anchor on "empty = −1, leaf = 0."
- **Complete ≠ full ≠ perfect** — interviewers probe this; a complete tree need not be full (last node may be a lone left child).
- **Array layout for sparse trees** silently explodes memory; it's a heap-only trick.

## See Also
- [[binary-search-trees]] — the ordered specialization; where the balance problem bites
- [[tree-traversals]] — DFS/BFS mechanics and their `O(h)` stack cost
- [[heaps]] — the complete-tree + array representation in action
- [[recursion]] — trees are the canonical recursive data structure
- [[avl-trees]] — enforcing `h = O(log n)` explicitly
- [[07-performance-engineering/data-locality|Data Locality]] — why pointer-chasing trees miss cache
