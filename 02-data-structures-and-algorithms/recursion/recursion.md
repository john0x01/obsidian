# Recursion

Recursion is a function defined in terms of itself: it solves a problem by reducing it to smaller instances of the same problem until it reaches a case small enough to answer directly. It is less a performance tool than a *modeling* tool — it lets code mirror the recursive structure of the data (trees, grammars, nested state) so directly that the solution often reads like the definition of the problem.

## Call-Stack Mechanics

Every recursive call is an ordinary function call, so it pushes a **stack frame** onto the call stack: the return address, the arguments, and local variables for that invocation. Frames stack up as the recursion descends and pop as each call returns, unwinding back to the caller. The pending computation *after* a recursive call (e.g. the `n *` below) lives in the parent frame, waiting.

```ts
function fact(n: number): number {
  if (n <= 1) return 1;          // base case — stops the descent
  return n * fact(n - 1);        // recursive case — n * (pending work) held on the stack
}
```

The maximum stack depth equals the recursion depth, so **recursion consumes O(depth) stack space** even when it uses no explicit data structure — a hidden space cost the Big-O of the *time* often obscures (see [[space-complexity]]).

## Base Case and Recursive Case

Correct recursion needs two things, and both failure modes are classic bugs:

- A **base case** that returns without recursing — omit it (or make it unreachable) and you recurse forever until the stack overflows.
- A **recursive case** that makes *strictly measurable progress* toward the base case. If the argument doesn't shrink toward the base on every path, termination is not guaranteed.

Reason about correctness by **structural induction**: assume the recursive calls return correct answers for smaller inputs (the *inductive hypothesis*), then check the combining step is right. You don't trace the whole stack in your head.

## The Recursion Tree

Model a recursion by its **recursion tree**: nodes are calls, children are the calls they make. The tree's *depth* is stack usage; the *total node count* is the running time. Linear recursion (`fact`) is a single spindly path — depth n, n nodes, O(n). Branching recursion fans out: naive Fibonacci makes two calls per node, producing ~2^n nodes and exponential time despite trivial per-call work. That same tree is the object you sum in a [[recurrence-relations|recurrence relation]] and the picture behind the Master Theorem in [[divide-and-conquer]].

## Direct, Indirect, and Mutual

- **Direct** — a function calls itself.
- **Indirect** — `f` calls `g` which calls `f`.
- **Mutual** — two (or more) functions call each other by design, e.g. `isEven`/`isOdd`, or recursive-descent parsers with a function per grammar rule. All share the same stack-frame mechanics.

## Tail Recursion and TCO

A call is in **tail position** if it is the *last* action of the function — nothing pending after it returns. A tail-recursive `fact` carries the running product in an accumulator so no multiply waits on the stack:

```ts
function fact(n: number, acc = 1): number {
  if (n <= 1) return acc;
  return fact(n - 1, acc * n);   // tail call — parent frame has nothing left to do
}
```

Because the parent frame is dead, a compiler can perform **tail-call optimization (TCO)**: reuse the current frame instead of pushing a new one, running the recursion in O(1) stack. **The senior caveat for our track: JavaScript engines generally do NOT implement TCO.** Proper tail calls are in the ES2015 spec, but only JavaScriptCore/Safari shipped them; V8 and SpiderMonkey do not. So in Node/Chrome, deep tail recursion **still overflows the stack** — writing it tail-recursively buys you nothing here. See [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]] for the language-level treatment.

## Converting Recursion ↔ Iteration

Any recursion can be made iterative by managing an **explicit stack** on the heap instead of the call stack:

```ts
function dfs(root: Node) {
  const stack = [root];               // heap stack replaces call frames
  while (stack.length) {
    const node = stack.pop()!;
    visit(node);
    for (const c of node.children) stack.push(c);
  }
}
```

This trades elegance for **robustness**: heap memory is far larger than the ~1 MB call stack, so it sidesteps stack-overflow on deep inputs, and it makes the state explicit. This is exactly how you make a recursive [[tree-traversals|tree traversal]] or graph DFS safe on adversarially deep inputs.

## When Recursion Clarifies (and When Not)

Reach for recursion when the *data or algorithm is itself recursive*: [[tree-traversals|tree traversals]], [[divide-and-conquer]] (mergesort, quicksort), [[backtracking]] (permutations, N-queens, parsing). Avoid it when the problem is naturally linear (a simple loop is clearer and cheaper) or when depth is unbounded/large in a non-TCO language — convert to iteration.

## Senior Pitfalls

- **Stack overflow on deep/adversarial input** — an unbalanced tree or a linked list of length 10⁶ blows a recursive traversal in JS; use an explicit stack.
- **Exponential re-computation** from overlapping subproblems (naive Fibonacci) — memoize to collapse the tree to a DAG; this is the doorway to [[dynamic-programming]] and [[07-performance-engineering/memoization|Memoization]].
- **Assuming TCO exists** — see above; it usually doesn't in JS.
- **Forgetting recursion's O(depth) space** when comparing against an iterative alternative that uses O(1) — it's not free.

## See Also
- [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]] — TCO and language semantics
- [[divide-and-conquer]] — recursion that splits and recombines
- [[backtracking]] — recursion that explores and undoes
- [[tree-traversals]] — the canonical recursive walk
- [[space-complexity]] — the hidden O(depth) stack cost
- [[recurrence-relations]] — solving the recursion tree's cost
