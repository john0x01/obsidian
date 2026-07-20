# Tree Traversals

A traversal is a systematic visit of every node in a [[binary-trees|tree]] exactly once. The order you visit in is not cosmetic — it determines what the traversal *computes*: in-order over a [[binary-search-trees|BST]] yields sorted output, post-order lets you free or evaluate a node only after its children, and pre-order gives you a serialization you can rebuild from. Choosing the wrong order is a correctness bug, not a style choice.

## DFS: pre-, in-, post-order

Depth-first search plunges down one branch before backtracking. The three variants differ only in **when you "visit" the node relative to recursing into its children**:

- **Pre-order** — node, left, right.
- **In-order** — left, node, right.
- **Post-order** — left, right, node.

```ts
function inorder(n: Node | null, out: number[]): void {
  if (!n) return;
  inorder(n.left, out);
  out.push(n.val);      // move this line up/down for pre-/post-order
  inorder(n.right, out);
}
```

**Iterative with an explicit stack.** Recursion uses the call stack implicitly; to bound memory (or dodge stack overflow on a deep/degenerate tree) you manage a [[stacks|stack]] yourself:

```ts
function inorderIter(root: Node | null): number[] {
  const out: number[] = [], st: Node[] = [];
  let cur = root;
  while (cur || st.length) {
    while (cur) { st.push(cur); cur = cur.left; } // push leftmost spine
    cur = st.pop()!;
    out.push(cur.val);                            // visit on pop
    cur = cur.right;
  }
  return out;
}
```

Pre-order iterative is the easiest (push right then left). Post-order iterative is the fiddly one: either use two stacks, or track a `lastVisited` pointer so you emit a node only after its right subtree is done. This mirrors the recursion-to-iteration conversion in [[recursion]].

## BFS: level-order

Breadth-first visits level by level using a FIFO [[queues|queue]] instead of a stack — the same frontier mechanism as [[graph-traversal]], since a tree is just an acyclic graph:

```ts
function levelOrder(root: Node | null): number[][] {
  const out: number[][] = []; if (!root) return out;
  const q: Node[] = [root];
  while (q.length) {
    const level: number[] = [], n = q.length;
    for (let i = 0; i < n; i++) {           // snapshot count = one level
      const node = q.shift()!;
      level.push(node.val);
      if (node.left)  q.push(node.left);
      if (node.right) q.push(node.right);
    }
    out.push(level);
  }
  return out;
}
```

The "snapshot `q.length` before the inner loop" trick is what separates levels cleanly.

## When each order matters

- **In-order** → sorted keys from a BST; also the basis for finding the k-th smallest and validating BST order.
- **Post-order** → any computation where a node depends on its children being resolved first: freeing/deleting a tree, evaluating an expression tree, computing subtree sizes/heights, bottom-up DP on trees.
- **Pre-order** → serialization, deep-copying, and prefix-expression emission — you record a node before its subtrees so a decoder can rebuild top-down.
- **BFS/level-order** → shortest path in edges (unweighted), level-by-level processing, "right side view," and anything where proximity to the root is the ordering you care about.

## Complexity

Every traversal visits each node once with `O(1)` work per node, so all are **`Θ(n)` time**. Space is the interesting axis:

- Recursive/stack DFS uses `O(h)` auxiliary space — `O(log n)` balanced, but `O(n)` on a degenerate tree (and a genuine stack-overflow risk).
- BFS uses `O(w)` where `w` is the maximum level width — up to `~n/2` for the bottom level of a balanced tree. So on wide balanced trees BFS costs *more* memory than DFS.

## Morris traversal — O(1) space (briefly)

Morris in-order achieves **`O(1)` auxiliary space** (no stack, no recursion) by temporarily rewiring the tree: before descending left, it makes the current node the right child of its in-order predecessor (a "thread"), then removes the thread on the way back. Still `O(n)` time (each edge is traversed a constant number of times). The cost is that it **mutates the tree during traversal**, so it's unsafe under concurrency and awkward if visitors expect an immutable structure. Know it exists for the "traverse with constant space" curveball; rarely worth it in production.

## Senior pitfalls

- **Recursion depth** on skewed trees overflows the stack — the reason to keep an iterative version handy.
- **Post-order iterative** is where off-by-one and re-visit bugs live; the `lastVisited` guard is the fix.
- Pre-order + in-order together **can reconstruct** a tree, but pre-order alone (or level-order alone) cannot unless you also encode nulls.
- BFS with `Array.shift()` is `O(n)` per dequeue in JS — use a real ring-buffer queue for large inputs, or amortize with two stacks.

## See Also
- [[binary-trees]] — the structure being traversed and its height bounds
- [[binary-search-trees]] — where in-order = sorted output
- [[graph-traversal]] — DFS/BFS generalized to graphs (visited-set required)
- [[recursion]] — the implicit call stack behind DFS
- [[stacks]] — explicit stack for iterative DFS
- [[queues]] — FIFO frontier behind BFS
