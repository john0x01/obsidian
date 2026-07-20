# Data Structures & Algorithms Review Questions

Senior-level self-test for the DSA track. Questions only — answer from your own mental model, then check against the linked notes. Grouped by area. Prioritize *why the bound holds* and *when it breaks*, not memorized Big-O.

## Analysis & Foundations

1. State the formal definitions of O, Ω, and Θ. Is "worst case" the same axis as "the Big-O of an algorithm"? Explain why not.
2. Why is the amortized cost of `push` on a doubling array O(1)? Prove it with the accounting (banker's) and the potential methods. How is amortized different from average-case?
3. Where does a good amortized bound still hurt you in production, and how would you mitigate the single expensive operation?
4. State the Master Theorem's three cases and the regularity condition. Solve T(n) = 2T(n/2) + O(n), T(n) = 2T(n/2) + O(1), and T(n) = 7T(n/2) + O(n²). When does the theorem not apply?
5. Distinguish auxiliary space from total space. How much stack space does a recursion of depth d with in-frame arrays of size k use?
6. Give a concrete case where an O(n) algorithm beats an O(log n) one for realistic n. What does Big-O deliberately hide?

## Arrays & Strings

7. What growth factor should a dynamic array use, and what is the tension between 2× and 1.5×? Why can't append be worst-case O(1)?
8. Build a structure answering arbitrary range-sum queries in O(1). Extend it to 2D. How do difference arrays flip the cost model for range *updates*?
9. When is the two-pointer technique applicable, and what precondition does the converging variant require? Contrast it with the sliding window.
10. Write binary search with a loop invariant you can defend. Where is the classic integer-overflow bug, and how do lower_bound / upper_bound differ? Explain "binary search on the answer."
11. Derive Kadane's algorithm as a DP. How do you handle an all-negative array, and how does the circular-array variant work?
12. Explain KMP's failure function and why the total matching time is O(n + m). How does Rabin-Karp's rolling hash achieve expected O(n + m), and what is its worst case?

## Linked Lists

13. Give three operations where a linked list beats a dynamic array and three where it loses. Why does the cache make the asymptotic comparison misleading?
14. Prove Floyd's cycle detection terminates and that the slow/fast pointers meet. How do you then find the cycle's start node?
15. Reverse a singly linked list in place. State the loop invariant. What does the recursive version cost in space and why?

## Stacks & Queues

16. Implement a queue with two stacks. What is the amortized cost per operation and why?
17. How does a ring buffer distinguish "full" from "empty," and what are the two standard fixes?
18. What invariant does a monotonic stack maintain? Use it to solve next-greater-element and largest-rectangle-in-histogram in O(n). Why is it linear despite the inner loop?
19. Priority queue is an ADT — compare a binary heap, a sorted array, a balanced BST, and an unsorted list as backing implementations across insert / extract-min / peek.

## Hashing

20. Why is hash-table lookup O(1) *average* but O(n) *worst*? State the assumption the average relies on.
21. What properties make a good hash function, and why does the avalanche property matter? When must you reach for a keyed hash like SipHash?
22. Compare separate chaining with open addressing (linear/quadratic/double hashing). What is clustering, why do tombstones exist, and how do load-factor thresholds differ between the two schemes?
23. Explain a hash-flooding DoS attack end to end. What defends against it?
24. What is the difference in guarantees and iteration order between a `HashMap`, `LinkedHashMap`, and `TreeMap` (or JS `Map` vs a plain object)?

## Trees

25. When is a binary tree operation O(log n) and when is it O(n)? What does the array representation buy you and what does it cost?
26. Delete a node with two children from a BST. Why the predecessor/successor? Why does inserting sorted data degrade a plain BST, and how do self-balancing trees prevent it?
27. Give the recursive and iterative (explicit-stack) forms of the three DFS orders. Which order yields sorted output from a BST, and which is right for deleting/evaluating a tree? How does Morris traversal reach O(1) space?
28. Contrast AVL and red-black trees on height bound, rotations per update, and typical use. Why do standard libraries favor red-black?
29. Why is build-heap O(n) and not O(n log n)? Explain sift-down vs sift-up and where each is used. Is heapsort stable? In-place?
30. How does a trie give O(L) lookup independent of the number of keys, and when does it beat a hash map? What does a radix/Patricia trie compress?
31. Why do databases use B-trees / B+ trees instead of binary search trees on disk? What does high fan-out buy, and why link the leaves in a B+ tree?
32. What problem do segment trees and Fenwick trees solve that prefix sums cannot? What is lazy propagation, and what does the Fenwick tree trade for its smaller footprint?

## Graphs

33. Compare adjacency matrix vs adjacency list on space and on the cost of "is there an edge u→v?" vs "iterate v's neighbors." When is each right?
34. BFS and DFS are both O(V + E) — when must you use BFS specifically? Classify the edge types a DFS discovers and what a back edge implies.
35. Give both algorithms for topological sort. How does each detect that no valid ordering exists?
36. Why does Dijkstra require non-negative edge weights? State its complexity with a binary heap. When do you switch to Bellman-Ford or Floyd-Warshall, and what does A* add?
37. State the cut property and use it to justify Kruskal and Prim. Which do you pick for a sparse vs a dense graph?
38. How do path compression and union by rank get union-find to near-O(α(n))? What is α(n) here?
39. Contrast Tarjan's and Kosaraju's SCC algorithms. What is the condensation graph, and how does SCC detect a directed cycle?

## Algorithmic Paradigms

40. What structural difference between subproblems separates divide-and-conquer from dynamic programming?
41. State the two properties a problem needs for a greedy algorithm to be correct. Give a problem where greedy is optimal and a minimal counterexample where it fails. What is an exchange argument?
42. Compare memoization (top-down) and tabulation (bottom-up): recomputation, space, ordering, and stack risk. How do you compute a DP's complexity from its state space?
43. Design the state and transition for edit distance / LCS / 0-1 knapsack. How do you reduce each to O(1)-row or O(single-array) space?
44. What makes backtracking tractable versus brute force? Sketch the choose/explore/un-choose skeleton for N-Queens and explain the pruning.
45. Prove the Ω(n log n) lower bound for comparison sorts. Which sorts hit it, and how do counting/radix/bucket sort beat it — and under what assumptions?
46. Compare mergesort, quicksort, and heapsort on time (best/avg/worst), space, and stability. What pivot strategy defends quicksort's worst case? What do Timsort and introsort actually do?
47. When does recursion risk a stack overflow, and does your language do tail-call optimization? How do you convert a recursion to an explicit-stack iteration?

## Bit Manipulation & Advanced

48. Explain `n & (n - 1)`, testing for a power of two, and using XOR to find the single unpaired element. How do you enumerate all subsets with a bitmask? What breaks in JavaScript's 32-bit bitwise ops?
49. Design an LRU cache with O(1) get and put. Why are *both* a hash map and a doubly linked list required? How would LFU or CLOCK differ?
50. How does a skip list achieve expected O(log n) without rotations, and why is it attractive for concurrent access?
51. What guarantees does a Bloom filter give and give up? Derive the intuition for the optimal number of hash functions, and name a system that uses one to avoid disk reads.

## Sorting Algorithms

52. Fill in a comparison table from memory: bubble, selection, insertion, shell, merge, quick, heap, counting, radix, bucket — best/avg/worst time, space, stable?, in-place?
53. Selection sort and insertion sort are both O(n²) — when does each win, and why is insertion sort O(n) on nearly-sorted input while selection sort is Θ(n²) even on sorted input?
54. Why is merge sort Θ(n log n) in *all* cases while quicksort is only average-case Θ(n log n)? Derive both from their recurrences. What input triggers quicksort's Θ(n²), and how do randomized / median-of-three pivots and 3-way partitioning defend against it?
55. Which of these sorts are stable, and which are in-place? For quicksort and heapsort explain *why* they are not stable.
56. Why is building a heap O(n) rather than O(n log n)? Given heapsort matches merge/quicksort's Big-O and uses O(1) space, why is it usually slower in wall-clock time?
57. How do counting, radix, and bucket sort beat the Ω(n log n) comparison lower bound, and exactly what assumption does each make? Why must the per-digit sort in radix sort be stable?
58. State and prove the Ω(n log n) lower bound for comparison sorts.
59. What do Timsort and introsort actually do, and what real weakness of the textbook algorithms does each fix?

## Searching Algorithms

60. Organize search algorithms by the precondition they require (none / sorted / uniform / structural). What bound does each achieve and why?
61. When does linear search beat binary search in practice despite the worse Big-O?
62. Contrast jump, interpolation, exponential, and binary search: the split rule, the assumption, the complexity, and the situation each is designed for. When is interpolation search O(log log n) and when does it degrade to O(n)?
63. Why is exponential search the right tool for an unbounded or unknown-length sorted stream?
64. What are the two distinct uses of ternary search, and why does the three-way split do *more* comparisons than binary search on a sorted array?

## Problem-Solving Patterns

65. Solve the two-sum family two ways (one-pass hashing vs sorted two-pointer): compare time, space, and preconditions. How does it generalize to 3-sum / k-sum?
66. Give three ways to find the longest palindromic substring (DP, expand-around-center, Manacher) with their complexities. What is the center/radius mirror trick that makes Manacher O(n)?
67. What single step enables most interval problems, and how do you merge overlapping intervals? Describe the sweep-line approach to maximum overlap.
68. Find the k-th smallest element in expected O(n) with quickselect — what is its worst case and how does median-of-medians remove it? When would you use a size-k heap for top-K instead?
69. When is cyclic sort applicable, and how does it hit O(n) time / O(1) space? Sketch how it solves find-the-missing-number and find-the-duplicate without extra memory.
