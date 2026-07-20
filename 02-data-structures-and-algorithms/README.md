# 02 · Data Structures & Algorithms

The DSA canon for mastery, not syntax: how each structure and algorithm works, the invariant it maintains, its **precise** complexity (worst / average / amortized) and *why* the bound holds, the trade-offs against the alternatives, and the senior-level pitfalls. Snippets are TypeScript and illustrate mechanism, not full solutions.

Ordered as a curriculum: **analysis foundations → linear structures → hashing → hierarchical structures → graphs → algorithmic paradigms → advanced**.

[← Back to vault index](../README.md)

## Analysis & Foundations
- [Asymptotic Notation](analysis/asymptotic-notation.md) — Big-O / Ω / Θ, best/average/worst case
- [Amortized Analysis](analysis/amortized-analysis.md) — aggregate, accounting, potential; why doubling is O(1)
- [Recurrence Relations & the Master Theorem](analysis/recurrence-relations.md)
- [Space Complexity](analysis/space-complexity.md) — auxiliary vs total, call-stack space, in-place
- [Big-O vs Reality](analysis/big-o-vs-reality.md) — constant factors, cache, small-n

## Arrays & Strings
- [Dynamic Arrays](arrays-and-strings/dynamic-arrays.md) — resizable buffers, amortized doubling
- [Prefix Sums](arrays-and-strings/prefix-sums.md) — range queries, difference arrays
- [Two-Pointer](arrays-and-strings/two-pointer.md)
- [Sliding Window](arrays-and-strings/sliding-window.md)
- [Kadane's Algorithm](arrays-and-strings/kadanes-algorithm.md)
- [String Matching Algorithms](arrays-and-strings/string-algorithms.md) — KMP, Rabin-Karp

## Linked Lists
- [Linked Lists](linked-lists/linked-lists.md) — singly/doubly/circular, list vs array
- [Fast & Slow Pointers](linked-lists/fast-slow-pointers.md) — Floyd cycle detection
- [In-Place Reversal](linked-lists/in-place-reversal.md)

## Stacks & Queues
- [Stacks](stacks-and-queues/stacks.md)
- [Queues](stacks-and-queues/queues.md) — ring buffers, deques
- [Monotonic Stack & Queue](stacks-and-queues/monotonic-stack-and-queue.md)
- [Priority Queues](stacks-and-queues/priority-queues.md) — the ADT

## Hashing
- [Hash Tables](hashing/hash-tables.md) — average O(1), worst case, hash-flooding
- [Hash Functions](hashing/hash-functions.md)
- [Collision Resolution](hashing/collision-resolution.md) — chaining vs open addressing
- [Hash Maps & Sets](hashing/hash-maps-and-sets.md)

## Trees
- [Binary Trees](trees/binary-trees.md)
- [Binary Search Trees](trees/binary-search-trees.md)
- [Tree Traversals](trees/tree-traversals.md) — DFS pre/in/post + BFS
- [AVL Trees](trees/avl-trees.md)
- [Red-Black Trees](trees/red-black-trees.md)
- [Heaps](trees/heaps.md) — binary heap, heapify, d-ary/Fibonacci
- [Tries](trees/tries.md)
- [B-Trees & B+ Trees](trees/b-trees-and-b-plus-trees.md)
- [Segment Trees & Fenwick Trees](trees/segment-trees-and-fenwick.md)

## Graphs
- [Graphs](graphs/graphs.md) — terminology & representations
- [Graph Traversal (BFS & DFS)](graphs/graph-traversal.md)
- [Topological Sort](graphs/topological-sort.md)
- [Shortest Paths](graphs/shortest-paths.md) — Dijkstra, Bellman-Ford, Floyd-Warshall, A*
- [Minimum Spanning Tree](graphs/minimum-spanning-tree.md) — Kruskal, Prim
- [Union-Find (Disjoint Set)](graphs/union-find.md)
- [Strongly Connected Components](graphs/strongly-connected-components.md)

## Algorithmic Paradigms
- [Divide & Conquer](paradigms/divide-and-conquer.md)
- [Greedy Algorithms](paradigms/greedy-algorithms.md)
- [Dynamic Programming](paradigms/dynamic-programming.md) — memoization vs tabulation
- [Backtracking](paradigms/backtracking.md)
- [Recursion](recursion/recursion.md) — the underlying technique

## Sorting
- [Sorting Algorithms](sorting/sorting-algorithms.md) — comparison table, the n·log n lower bound, how to choose
- [Bubble Sort](sorting/bubble-sort.md)
- [Selection Sort](sorting/selection-sort.md)
- [Insertion Sort](sorting/insertion-sort.md)
- [Shell Sort](sorting/shell-sort.md)
- [Merge Sort](sorting/merge-sort.md)
- [Quick Sort](sorting/quick-sort.md)
- [Heap Sort](sorting/heap-sort.md)
- [Counting Sort](sorting/counting-sort.md)
- [Radix Sort](sorting/radix-sort.md)
- [Bucket Sort](sorting/bucket-sort.md)
- [Hybrid Sorts (Timsort & Introsort)](sorting/hybrid-sorts.md) — what real stdlibs use

## Searching
- [Searching Algorithms](searching/searching-algorithms.md) — the landscape & how to choose
- [Linear Search](searching/linear-search.md)
- [Binary Search](searching/binary-search.md) — invariants, and search-on-the-answer
- [Jump Search](searching/jump-search.md)
- [Interpolation Search](searching/interpolation-search.md)
- [Exponential Search](searching/exponential-search.md)
- [Ternary Search](searching/ternary-search.md)

## Problem-Solving Patterns
- [Two-Sum & Complement Hashing](patterns/two-sum-and-complement-hashing.md)
- [Palindrome Techniques](patterns/palindrome-techniques.md) — expand-around-center, Manacher
- [Interval Patterns](patterns/interval-patterns.md) — merge intervals, sweep line
- [Quickselect & Top-K](patterns/quickselect-and-top-k.md)
- [Cyclic Sort](patterns/cyclic-sort.md)

## Bit Manipulation
- [Bit Manipulation](bit-manipulation/bit-manipulation.md)

## Advanced Structures
- [LRU Cache](advanced/lru-cache.md)
- [Skip Lists](advanced/skip-lists.md)
- [Bloom Filters](advanced/bloom-filters.md)

## Self-Test
- [Review Questions](review-questions.md)
