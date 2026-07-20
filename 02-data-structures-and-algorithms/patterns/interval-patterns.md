# Interval Patterns

Interval problems — merging overlapping ranges, inserting a new range, counting the maximum simultaneous overlap (meeting rooms) — almost all yield to the same first move: **sort, then sweep**. Sorting turns a tangle of unordered `[start, end]` pairs into a sequence where each range only interacts with its neighbours, collapsing an apparent O(n²) pairwise problem into O(n log n).

## The overlap test

Two closed intervals `a` and `b` overlap iff `a.start <= b.end && b.start <= a.end`. Once sorted by start (`a.start <= b.start`), this simplifies: `b` overlaps the running interval iff `b.start <= a.end`. That single comparison drives merging.

```
[1,3] [2,6] overlap? 1<=6 && 2<=3 → yes, union = [1,6]
[1,3] [4,5] overlap? 4<=3 → no
```

## Merge overlapping intervals

Sort by start; keep a "current" interval; each incoming interval either extends it (overlap) or starts a fresh one.

```ts
type Interval = [number, number];

function merge(intervals: Interval[]): Interval[] {
  if (intervals.length === 0) return [];
  intervals.sort((a, b) => a[0] - b[0]); // sort by start — the enabling step
  const out: Interval[] = [intervals[0]];
  for (let i = 1; i < intervals.length; i++) {
    const last = out[out.length - 1];
    const [s, e] = intervals[i];
    if (s <= last[1]) {
      last[1] = Math.max(last[1], e); // overlap → extend the end
    } else {
      out.push([s, e]);               // disjoint → new block
    }
  }
  return out;
}
```

Why sorting works: after sorting by start, any interval that overlaps an earlier one must overlap the *most recent* merged block, because all earlier blocks have smaller-or-equal starts and we've already absorbed everything reachable. So one linear pass suffices.

## Dry run / trace (merge)

Input `[[1,3],[2,6],[8,10],[15,18]]` (already start-sorted):

| step | incoming | `last` | test `s <= last[1]` | `out` after |
|------|----------|--------|---------------------|-------------|
| init | — | — | — | `[[1,3]]` |
| 1 | `[2,6]` | `[1,3]` | `2 <= 3` ✓ | `[[1,6]]` (extend to max(3,6)) |
| 2 | `[8,10]` | `[1,6]` | `8 <= 6` ✗ | `[[1,6],[8,10]]` |
| 3 | `[15,18]` | `[8,10]` | `15 <= 10` ✗ | `[[1,6],[8,10],[15,18]]` |

Result: `[[1,6],[8,10],[15,18]]`.

## Insert interval

Given an already-sorted, non-overlapping list and a new interval, you can skip the full re-sort — walk once in three phases: (1) copy intervals ending before the new one starts, (2) merge everything that overlaps into the new interval, (3) copy the rest.

```ts
function insert(intervals: Interval[], nw: Interval): Interval[] {
  const out: Interval[] = [];
  let [s, e] = nw, i = 0;
  const n = intervals.length;
  while (i < n && intervals[i][1] < s) out.push(intervals[i++]);   // ends before new
  while (i < n && intervals[i][0] <= e) {                          // overlaps → absorb
    s = Math.min(s, intervals[i][0]);
    e = Math.max(e, intervals[i][1]);
    i++;
  }
  out.push([s, e]);
  while (i < n) out.push(intervals[i++]);                          // after new
  return out;
}
```

Because the input is presorted, this is **O(n)** — no sort needed.

## Sweep-line: maximum overlap

For "how many meetings run at once?" (or "minimum rooms needed"), don't track intervals — track **events**. Split each `[s, e]` into a `+1` at `s` and a `−1` at `e`, sort all endpoints, and scan keeping a running count; the peak is the answer.

```ts
function maxOverlap(intervals: Interval[]): number {
  const ev: [number, number][] = []; // [time, delta]
  for (const [s, e] of intervals) { ev.push([s, +1]); ev.push([e, -1]); }
  // tie-break: process end (-1) before start (+1) at equal times for closed-then-open
  ev.sort((a, b) => a[0] - b[0] || a[1] - b[1]);
  let cur = 0, best = 0;
  for (const [, delta] of ev) { cur += delta; best = Math.max(best, cur); }
  return best;
}
```

The sort of endpoints is the enabling step again; the scan is O(n). Equivalently, a two-heap / sorted-starts-vs-sorted-ends formulation gives the same answer using a [[priority-queues|min-heap]] of end times.

## Complexity

- **Merge / max-overlap**: dominated by the sort → **O(n log n) time**, **O(n) space** for the output/events (O(log n) stack if sorting in place).
- **Insert into presorted list**: **O(n) time**, O(n) output space — the whole point is that no sort is required.
- The linear scans themselves are O(n); sorting is the only super-linear cost, which is why "can I assume it's sorted?" is the key question to ask.

## Variants & trade-offs

- **Interval scheduling (max non-overlapping)**: a [[greedy-algorithms|greedy]] variant — sort by *end* and greedily take the earliest-finishing compatible interval.
- **Weighted interval scheduling**: greedy fails; needs [[dynamic-programming]] with binary search.
- **Many dynamic queries** (stabbing, overlap counts under updates): promote to a `segment-trees-and-fenwick` / interval tree rather than re-sorting each time.

## When to use / when not

Reach for sort-then-sweep whenever the data is a static set of ranges and you need merges, overlaps, or peak concurrency. Avoid re-sorting inside a loop — sort once. If intervals stream in or mutate frequently, a balanced-tree/heap structure amortizes better than repeated sorts.

## Interview pitfalls & gotchas

- **Sort by the wrong key**: merging needs sort-by-**start**; max-non-overlapping greedy needs sort-by-**end**. Mixing them silently produces wrong answers.
- **Closed vs half-open intervals**: does `[1,3]` touch `[3,5]`? With closed intervals `3 <= 3` counts as overlap; with half-open `[start, end)` it does not. This flips the `<=` vs `<` in both the merge test and the sweep tie-break — clarify before coding.
- **Sweep tie-break at equal times**: whether a meeting ending at `t` frees a room *before* one starting at `t` depends on open/closed semantics; get the secondary sort key right.
- **Mutating the shared `last` reference** in merge while also holding it in `out` is intentional here, but aliasing bugs are common if you copy carelessly.
- **Empty input** and single-interval edge cases.

## See Also

- [[sorting-algorithms]] — the enabling first step for nearly every interval problem
- [[greedy-algorithms]] — interval scheduling / earliest-finish selection
- [[priority-queues]] — heap formulation of meeting-rooms
- [[two-pointer]] — the merge scan is a one-pointer sweep
- [[segment-trees-and-fenwick]] — dynamic interval queries
