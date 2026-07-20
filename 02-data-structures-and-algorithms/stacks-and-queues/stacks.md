# Stacks

A stack is a **LIFO** (last-in, first-out) collection: you push onto the top and pop from the top, and that is the only access you get. This deliberate restriction is the feature — it models any process that must be *unwound in reverse order*, from expression parsing to the machine's own call stack.

## LIFO Semantics & the Invariant

Only the most recently added element is reachable. There is no indexing into the middle, no iteration guarantee beyond "top first." The invariant is a single top-of-stack marker; every operation touches only that end, which is exactly why all core operations are O(1).

## Core Operations — All O(1)

- **push(x)** — add to the top.
- **pop()** — remove and return the top (error/underflow on empty).
- **peek()** — inspect the top without removing.

```ts
class Stack<T> {
  private a: T[] = [];
  push(x: T) { this.a.push(x); }
  pop(): T | undefined { return this.a.pop(); }   // O(1) amortized
  peek(): T | undefined { return this.a[this.a.length - 1]; }
  get size() { return this.a.length; }
}
```

## Implementations

- **Array-backed** — a [[dynamic-arrays|dynamic array]] where the top is the last index. `push`/`pop` are O(1) *amortized*; a push occasionally triggers an O(n) resize, but doubling makes the average O(1) (see [[amortized-analysis]]). This is the default: contiguous memory means excellent cache locality and tiny constants.
- **Linked-list-backed** — push/pop at the head of a singly [[linked-lists|linked list]]. Every operation is *worst-case* O(1) with no resize spikes, which matters for hard real-time bounds, but each node is a separate heap allocation with a pointer, so it chases the cache and pressures the GC. Prefer arrays unless you need guaranteed per-op latency.

The trade-off mirrors the general array-vs-list story: amortized-fast-and-local versus worst-case-bounded-but-scattered.

## Relationship to the Call Stack

The program **call stack** *is* a stack: each function call pushes a stack frame (return address, saved registers, locals, arguments); returning pops it. This is why deep or unbounded [[recursion]] causes a **stack overflow** — the frames are LIFO and the region is bounded. It's also the mechanical reason recursion and an explicit stack are interchangeable: any recursive algorithm can be rewritten iteratively by managing your own stack, trading implicit frames for heap-allocated state you control (and can grow past the OS stack limit). See [[01-programming-foundations/paradigms/recursion-and-tail-calls|recursion and tail calls]].

## Canonical Patterns

- **Balanced parentheses / bracket matching** — push each opener, pop and check the match on each closer; valid iff the stack empties exactly. The stack captures the *nesting* structure that a flat counter cannot (a counter can't tell `([)]` is invalid).
- **Expression evaluation / RPN** — evaluate postfix by pushing operands and, on each operator, popping the operands and pushing the result. The **shunting-yard** algorithm uses an operator stack to convert infix to postfix while respecting precedence and associativity.
- **Iterative DFS** — replace the recursion stack with an explicit one to traverse a [[graphs|graph]] or tree without risking overflow on deep inputs; see [[graph-traversal]]. Note iterative DFS visits in a slightly different order than recursive DFS unless you're careful about push order.
- **Backtracking** — the implicit call stack records the path of choices so you can undo the last decision and try the next; explicit-stack variants make this concrete. See [[backtracking]].
- **Undo/redo** — two stacks; each action pushes its inverse. Also the substrate for the [[monotonic-stack-and-queue|monotonic stack]], where an ordering invariant on the stack yields amortized-O(n) next-greater-element and histogram algorithms.

## Complexity

push, pop, peek, size: **O(1)** (amortized O(1) for array push under doubling; worst-case O(1) for the linked version). Space is O(n) for n elements. Iterating/searching for a specific value is O(n) and defeats the purpose — if you need that, you picked the wrong structure.

## Senior Pitfalls

- **Underflow** — popping/peeking an empty stack; decide between returning a sentinel and throwing, and be consistent. Many parser bugs are unchecked empties.
- **Confusing amortized with worst-case** — the array push is amortized O(1); a single push during a resize is O(n). In a latency-sensitive loop, prefer the linked implementation or pre-size the array.
- **Stack overflow from recursion** — deep DFS on adversarial input (a degenerate long path) crashes; convert to an explicit stack for unbounded depth.
- **Growth direction on raw arrays** — always push/pop at the *end* of a dynamic array; treating index 0 as the top makes every op O(n) from shifting.
- **Order in iterative DFS** — to mimic recursive left-to-right DFS with a stack, push children in reverse so the leftmost pops first.

## See Also
- [[queues]] — the FIFO counterpart; contrast the access discipline
- [[recursion]] — the call stack is a stack; recursion ↔ explicit stack
- [[graph-traversal]] — iterative DFS uses an explicit stack
- [[monotonic-stack-and-queue]] — a stack with an ordering invariant
- [[dynamic-arrays]] — the usual backing store
- [[backtracking]] — the call stack records the choice path
