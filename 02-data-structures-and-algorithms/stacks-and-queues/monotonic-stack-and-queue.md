# Monotonic Stack & Queue

A monotonic stack (or deque) is an ordinary [[stacks|stack]]/[[queues|deque]] that maintains an extra **ordering invariant**: its contents are kept sorted (increasing or decreasing) by evicting elements that violate the order before pushing a new one. That single discipline turns a family of "for each element, find the nearest larger/smaller neighbour" problems from O(n²) brute force into **amortized O(n)**.

## The Monotone Invariant

Keep the stack monotonic — say **strictly decreasing** from bottom to top. Before pushing `x`, pop every top element that `x` would violate (here, every element `≤ x`). The elements you pop are precisely those for which `x` is the answer to "next greater element," and whatever remains on top after pushing is `x`'s own "previous greater element." The invariant *is* the algorithm: each pop and each survivor encodes a resolved relationship.

```ts
// Next Greater Element to the right, for each index
function nextGreater(nums: number[]): number[] {
  const res = new Array(nums.length).fill(-1);
  const st: number[] = []; // indices, values strictly decreasing
  for (let i = 0; i < nums.length; i++) {
    while (st.length && nums[st[st.length - 1]] < nums[i]) {
      res[st.pop()!] = nums[i];      // nums[i] resolves the popped index
    }
    st.push(i);
  }
  return res; // indices left on the stack have no greater element → -1
}
```

Store **indices**, not values, so you can also recover distances and widths.

## Why Amortized O(n)

The inner `while` looks like it could make this O(n²), but count the *total* work across the whole run, not per iteration: **each index is pushed exactly once and popped at most once**, so the combined number of push/pop operations is at most `2n`. The outer loop runs `n` times; the amortized cost per element is O(1). This is a textbook [[amortized-analysis|aggregate-method]] argument — the occasional long pop chain is paid for by the many elements that never trigger one. Total: **O(n) time, O(n) space** for the stack.

## Next-Greater / Next-Smaller

Choosing the invariant direction and the pop comparator picks the query:

- **Next greater to the right** → decreasing stack, pop while `top < x`.
- **Next smaller to the right** → increasing stack, pop while `top > x`.
- **Previous greater/smaller** → the element left on the stack when you push `x`.
- For a **circular** array (wrap-around "next greater"), iterate `2n` times using modulo indexing and only push during the first pass.

Handle ties deliberately: `<` vs `≤` decides whether equal elements block or pass, which matters for "strictly greater" vs "greater-or-equal" semantics.

## Largest Rectangle in a Histogram

Maintain a stack of bar indices with **increasing** heights. When a shorter bar arrives, pop each taller bar and finalize the rectangle for which that popped bar was the limiting height: its width spans from the new bar's left edge to the index just after the previous element still on the stack. Every bar is pushed and popped once → **O(n)**, versus the O(n²) "expand from each bar" approach. A sentinel `0`-height bar appended at the end flushes the stack cleanly. The same skeleton solves "trapping rain water" and the maximal all-ones submatrix (histogram per row).

## Sliding-Window Maximum via a Monotonic Deque

To report the maximum of every window of size `k` in O(n), keep a **deque of indices** whose values are decreasing front-to-back:

```ts
function maxSlidingWindow(nums: number[], k: number): number[] {
  const dq: number[] = [], out: number[] = []; // dq: decreasing values
  for (let i = 0; i < nums.length; i++) {
    while (dq.length && nums[dq[dq.length - 1]] <= nums[i]) dq.pop();
    dq.push(i);
    if (dq[0] <= i - k) dq.shift();            // drop indices left of window
    if (i >= k - 1) out.push(nums[dq[0]]);     // front is the max
  }
  return out;
}
```

Two evictions keep it correct: pop from the **back** any element smaller than the incoming one (it can never be a future maximum while the newcomer lives), and drop from the **front** any index that has scrolled out of the window. The front is always the current window's maximum. Each index enters and leaves the deque once → **O(n) time, O(k) space** — strictly better than a heap-based O(n log k) [[sliding-window]] solution.

## Senior Pitfalls

- **Storing values instead of indices** — you lose the ability to compute widths and to expire out-of-window elements.
- **Wrong strictness (`<` vs `≤`)** — silently mishandles duplicates; pin it to the exact "greater" vs "greater-or-equal" requirement.
- **Forgetting the flush** — elements left on the stack at the end still need finalizing (histogram) or a `-1` default (next-greater); a sentinel automates this.
- **Reaching for a heap** — for sliding-window *extrema* the monotonic deque is O(n) and simpler; a [[priority-queues|priority queue]] gives O(n log k) and needs lazy deletion.
- **Deque end confusion** — push/pop the *back* for the monotone invariant, pop the *front* for window expiry; mixing them corrupts both.

## See Also
- [[stacks]] — the underlying LIFO structure and its O(1) ops
- [[queues]] — the deque that backs sliding-window maximum
- [[sliding-window]] — the window framework this optimizes
- [[priority-queues]] — the heap-based alternative for window extrema
- [[amortized-analysis]] — why each-element-once yields O(n)
