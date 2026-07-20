# In-Place Linked-List Reversal

Reversing a linked list in place flips the direction of every `next` pointer without allocating a new list. It is the canonical exercise in **careful pointer rewiring under an invariant**, and it generalizes to reversing sublists and fixed-size groups — a building block for problems from "reverse nodes in k-groups" to LRU reordering.

## The Iterative Mechanism & Invariant

Walk the list with three cursors — `prev`, `curr`, `next`. The **invariant**: everything from `prev` backward is already reversed and correctly linked; `curr` is the first not-yet-processed node. Each step detaches `curr`, points it back at `prev`, and slides the window forward.

```ts
function reverse<T>(head: Node<T> | null): Node<T> | null {
  let prev: Node<T> | null = null;
  let curr = head;
  while (curr) {
    const next = curr.next; // 1. SAVE before we clobber it
    curr.next = prev;       // 2. flip the pointer backward
    prev = curr;            // 3. advance prev
    curr = next;            // 4. advance curr
  }
  return prev;              // new head = last node visited
}
```

The single most important line is step 1. `curr.next = prev` destroys the only forward reference, so you **must** cache `next` first — this is the "lost-pointer" hazard from [[linked-lists]] in its purest form. When the loop ends, `curr` is null and `prev` holds the old tail, which is the new head.

Trace `1 → 2 → 3`:

```
prev  curr        after step
null   1  →  null<-1   2 3
  1    2  →  null<-1<-2   3
  2    3  →  null<-1<-2<-3
```

## Recursive Reversal & Its Stack Cost

The recursive form reverses the tail first, then fixes the two-node link on the way back up:

```ts
function reverseR<T>(head: Node<T> | null): Node<T> | null {
  if (!head || !head.next) return head;   // base: 0 or 1 node
  const newHead = reverseR(head.next);     // reverse the rest
  head.next.next = head;                   // make successor point back
  head.next = null;                        // sever old forward link
  return newHead;                          // propagated unchanged
}
```

It is elegant but **O(n) space** in call-stack frames versus O(1) for the iterative version, and a long list will **overflow the stack** — a real production hazard in languages without tail-call elimination (JS engines generally do not apply TCE here, and this call isn't in tail position anyway). See [[01-programming-foundations/paradigms/recursion-and-tail-calls|recursion and tail calls]]. For anything unbounded, prefer the iterative loop.

## Reversing a Sublist & K-Groups

Real problems rarely reverse the whole list. Two extensions matter:

- **Reverse between positions `[m, n]`** — walk to the node before `m`, reverse exactly `n − m + 1` nodes with the three-pointer loop, then stitch three connections: the pre-`m` node's `next` to the new sub-head, and the old sub-head's `next` to the node after `n`. A **dummy head** removes the edge case where `m = 1` (reversing from the real head), because you always have a predecessor to reattach.
- **Reverse in k-groups** — repeatedly reverse `k` nodes at a time, leaving a trailing group of fewer than `k` untouched (or reversing it, per spec). The trap is bookkeeping: each reversed block's new tail must link to the next block's eventual new head. Verify a full group of `k` exists *before* reversing it, or you'll partially reverse the remainder.

```ts
// after reversing a k-block, connect: prevGroupTail.next = newHead;
//                                     oldHead.next = nextGroupHead;
```

## Complexity

Iterative reversal is **O(n) time, O(1) space** — each node's pointer is rewritten exactly once. Recursive reversal is O(n) time but **O(n) space** from the frames. Sublist and k-group variants remain O(n) time overall: every node is touched a constant number of times regardless of group size, so k-groups is O(n), not O(n·k).

## Senior Pitfalls

- **Forgetting to save `next`** before overwriting `curr.next` — the defining bug; the list truncates to one node.
- **Returning `head` instead of `prev`** — the old head is now the tail.
- **Broken stitch on sublists/k-groups** — mislinking the boundary nodes silently drops or duplicates a segment; sketch the three reconnections explicitly.
- **Recursion depth** — reversing a million-node list recursively is a crash, not a slowdown.
- **Doubly linked lists** — you must also swap each node's `prev`, or reverse by simply swapping the list's head/tail pointers and treating `prev`/`next` symmetrically.

## See Also
- [[linked-lists]] — node layout and the lost-pointer hazard
- [[recursion]] — the recursive form and why its stack cost bites
- [[fast-slow-pointers]] — pairs with reversal for palindrome checks
- [[01-programming-foundations/paradigms/recursion-and-tail-calls|Recursion & Tail Calls]] — why the recursive version isn't O(1) space
