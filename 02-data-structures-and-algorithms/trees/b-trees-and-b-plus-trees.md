# B-Trees & B+ Trees

A **B-tree** is a balanced, high-fan-out search tree designed for the **external-memory (block) model**: data lives on disk or SSD and the cost that matters is the number of *page transfers*, not the number of comparisons. By packing many keys into each node and keeping the tree shallow, a B-tree turns a lookup over billions of records into a handful of I/Os. It is the structure behind almost every relational-database index and filesystem.

## Why Binary Trees Fail on Disk

The [[07-performance-engineering/data-locality|memory hierarchy]] spans enormous latency gaps: an L1 hit is ~1 ns, a random disk seek can be ~10 ms — a factor of millions. Storage is read in fixed **pages/blocks** (often 4–16 KB), and one page fetch costs the same whether you use one byte of it or all of it. A [[binary-search-trees|binary BST]] of a billion keys has height ~30, and each node is one pointer-chase — potentially **30 random I/Os** per lookup. That is fine in RAM (a node fits a cache line) and catastrophic on disk.

The fix is to make the branching factor huge so the tree is short. The block model rewards this directly: height drives I/O count.

## The Idea and Invariants

A **node is a disk page.** Choose the order (max children) so a full node fills one page — thousands of keys per node is normal. Within a node, keys are sorted and interleaved with child pointers; a search does a binary search *inside* the page (cheap, in RAM) then follows one pointer to the next page.

For a B-tree of order m (up to m children, m−1 keys):

- Every node except the root holds between `⌈m/2⌉−1` and `m−1` keys (at least **half full**) — this bounds height and guarantees space efficiency.
- Keys within a node are sorted; a node with k keys has k+1 children spanning the gaps between them.
- **All leaves lie at the same depth.** The tree is perfectly height-balanced at all times.

Height is `≈ log_B n` where B is the fan-out. With B = 1000 and n = 10⁹, height is ~3 — so ~3 page reads (and the top levels stay cached), versus 30 for a binary tree.

**Split/merge keep the balance.** On insert, descend to a leaf and add the key; if the node overflows (m keys), **split** it in half and push the median key up into the parent. A split can cascade upward; when it reaches and splits the root, the tree grows one level — the *only* way its height increases, which is why leaves stay equidepth. On delete, if a node underflows below the minimum, it **borrows** a key from a sibling (rotating through the parent) or **merges** with a sibling and pulls a separator key down — the mirror operation, which can cascade and shrink height.

## B+ Trees

A **B+ tree** refines the layout for range access and is what production databases actually use:

- **All records/values live only in the leaves.** Internal nodes store *only keys as separators* to route searches — no data. Because routers are smaller, each internal node fits even more keys, raising fan-out and further flattening the tree.
- **Every search descends to a leaf**, so all lookups cost the same depth (a nice property for predictable latency).
- **Leaves are linked** in a (usually doubly) linked list. Once you find the start of a range, you scan sideways leaf-to-leaf without ever walking back up — making `WHERE age BETWEEN 20 AND 40`, `ORDER BY`, and full scans efficient sequential reads.

```
        [ 30 | 70 ]              internal: separator keys only
       /     |     \
  [10 20]  [30 50] [70 90]       leaves hold the actual rows
     └──────►└──────►└──────►     leaves linked for range scans
```

These are exactly the trade-offs a storage engine cares about: see [[04-databases/b-trees|B-Trees (Databases)]] and how they power [[04-databases/indexing|Indexing]].

## Complexity

- **Search / insert / delete:** O(log_B n) page accesses — the height. In comparison terms it is still O(log n), but the base B and the I/O-count framing are the point.
- **Range query of r results:** O(log_B n + r/B) — find the start, then stream linked leaves.
- **Space:** O(n), with nodes guaranteed ≥ half full so utilization stays high.

## Trade-offs and Pitfalls

- **B-tree vs B+ tree:** a plain B-tree can store values in internal nodes and occasionally return early on a lookup, but its range scans are clumsy (in-order traversal bouncing up and down). B+ trees trade that for uniform depth and linked-leaf scans — almost always the better on-disk choice.
- **Write amplification & fragmentation:** random inserts cause splits and page rewrites. Write-heavy workloads increasingly favor [[04-databases/lsm-trees|LSM-trees]], which batch writes sequentially at the cost of read amplification.
- **Concurrency:** real engines use latch-crabbing / B-link trees so readers and writers don't block on whole subtrees — a detail the textbook invariant hides.
- **Don't tune B by comparison count.** The right order is dictated by page size and key size, i.e. what makes a node fill one block — an [[07-performance-engineering/big-o-in-practice|architecture-aware]] decision, not an asymptotic one.

## See Also
- [[04-databases/b-trees|B-Trees (Databases)]] — the storage-engine perspective
- [[04-databases/indexing|Indexing]] — how indexes are built on B+ trees
- [[binary-search-trees]] — the in-memory counterpart that fails on disk
- [[04-databases/lsm-trees|LSM-Trees]] — the write-optimized alternative
- [[avl-trees]] — balanced BSTs for RAM-resident data
