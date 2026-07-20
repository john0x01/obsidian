# Fast & Slow Pointers

Fast & slow pointers (Floyd's "tortoise and hare") walk one traversal with two cursors moving at different speeds. Their changing gap lets you detect a **cycle**, find its **start**, or locate the **midpoint** / **nth-from-end** node — all in a single O(n) pass using O(1) extra space, without a visited-set or a second traversal.

## The Idea

Advance `slow` by one node and `fast` by two per step. If the list is acyclic, `fast` reaches null and you're done. If there is a cycle, `fast` laps `slow` and they eventually collide inside the loop — because once both are in the cycle, the gap between them shrinks by exactly one each step and therefore reaches zero.

```ts
function hasCycle<T>(head: Node<T> | null): boolean {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow!.next;        // +1
    fast = fast.next.next;    // +2
    if (slow === fast) return true;
  }
  return false;               // fast hit null → no cycle
}
```

## Why They Meet — and Where the Cycle Starts

Say the tail-to-cycle-entry distance is `μ` and the cycle length is `λ`. Once both pointers are inside the cycle, consider their positions modulo `λ`. Each step, `fast` gains one on `slow`, so the distance closes by 1 per step and hits 0 within `λ` steps — **meeting is guaranteed**.

Finding the entry uses a distance argument. When they meet, `slow` has travelled `d` and `fast` has travelled `2d`; the extra `d = 2d − d` is a whole number of loops, so `d` is a multiple of `λ`. `slow` sits `d` steps into the list, i.e. `d − μ` steps past the entry. Now reset one pointer to the head and advance **both at speed one**: the head pointer needs `μ` steps to reach the entry, and the other — already `d − μ = (kλ) − μ` past the entry — also lands on the entry after `μ` steps (because it wraps a whole number of loops). They collide **exactly at the cycle start**.

```ts
function cycleStart<T>(head: Node<T> | null): Node<T> | null {
  let slow = head, fast = head;
  while (fast && fast.next) {
    slow = slow!.next; fast = fast.next.next;
    if (slow === fast) {                 // meeting point
      let p = head;
      while (p !== slow) { p = p!.next; slow = slow!.next; }
      return p;                          // the entry node
    }
  }
  return null;
}
```

The cycle length itself falls out for free: from the meeting point, walk `slow` around until it returns — that many steps is `λ`.

## Midpoint & Nth-From-End in One Pass

The same speed trick answers positional queries without first computing the length:

- **Midpoint** — when `fast` reaches the end, `slow` is at the middle. The stop condition decides the tie: `while (fast && fast.next)` leaves `slow` on the **first** of two middles; `while (fast.next && fast.next.next)` leaves it on the second. Get this wrong and you split a list one node off. This midpoint find is the setup step for merge-sorting a list and for palindrome checks.
- **Nth-from-end** — give `fast` an `n`-node head start, then advance both by one until `fast` hits the end; `slow` now trails by `n` and sits on the target. This is the standard "remove Nth from end" pattern and needs a dummy head to handle removing the actual head cleanly.

## Complexity & Trade-offs

Time is O(n): even with a cycle, the pointers meet within `μ + λ ≤ n` steps, and the entry-finding phase is another ≤ n. Space is **O(1)** — the whole point. The obvious alternative, a hash set of visited nodes, is also O(n) time but O(n) space and thrashes the cache; Floyd's wins on memory and locality. The gap-based `n`-ahead idea generalizes the same-direction [[two-pointer]] technique from arrays to lists.

## Senior Pitfalls

- **Null-deref on `fast.next.next`** — always guard `fast && fast.next` *before* the double hop; the order of the checks matters.
- **Meeting point ≠ cycle start** — a common bug is returning the collision node as the entry. You must run the reset-to-head phase.
- **Off-by-one midpoint** — pick the loop condition to match whether you want the first or second middle, and be deliberate about even-length lists.
- **Assuming a meeting proves list length** — it proves a cycle exists; the length needs the extra walk.
- Works on any "successor function" space (e.g. `x → f(x)` for duplicate-number problems), not just physical linked lists.

## See Also
- [[linked-lists]] — the structure these pointers traverse
- [[two-pointer]] — the array-side sibling of the same-direction variant
- [[in-place-reversal]] — pairs with midpoint-finding for palindrome checks
- [[recursion]] — the recursive alternative for list splitting
