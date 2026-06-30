# Lanes And Priorities

**Lanes** are React's priority model for updates, introduced to replace the older `expirationTime` number. A lane is a single bit in a 31-bit bitmask; a *set* of lanes is just a bitwise-OR of those bits. The win over a scalar priority is that bitmasks let React represent and manipulate **arbitrary, non-contiguous groups of priorities** with cheap integer ops — testing membership (`&`), merging (`|`), and removing (`& ~`) — which a single ordered number cannot express.

## Why A Bitmask

A scalar "this update has priority N" forces a total order and can't say "these three unrelated updates should be processed together but not with that fourth one." Lanes can. React reserves ranges of bits for categories, roughly highest-to-lowest urgency:

- `SyncLane` — discrete, synchronous (e.g. the click that must feel instant).
- `InputContinuousLane` — continuous input like dragging/hover.
- `DefaultLane` — ordinary updates (a `setState` outside any special context).
- `TransitionLanes` — a *range* of ~16 lanes for `startTransition` work.
- `RetryLanes` — Suspense retry attempts.
- `Idle` / `OffscreenLane` — lowest priority, deferrable indefinitely.

A higher (numerically lower-bit) lane is more urgent. React picks the **highest-priority set of lanes** pending on the root and renders exactly those, leaving lower lanes for a later pass.

## Assigning A Lane

When an update is dispatched, React asks "what is the current priority context?" via `requestUpdateLane`:

- Inside an event handler, the lane derives from the event type (a click → `SyncLane`/discrete; scroll → continuous).
- Inside `startTransition`, React sets a flag so the update gets a `TransitionLane` (see [[transitions]]).
- `useDeferredValue` produces a deferred, lower-priority update.
- Outside any of these, it falls to `DefaultLane`.

The lane is OR-ed into the fiber's `lanes`, and propagated up every `return` pointer into each ancestor's `childLanes`, so the root knows a subtree has pending work without walking the whole tree. The Scheduler is then asked to call React back at a priority translated from the chosen lanes (see [[work-loop-and-scheduling]]).

## Batching

All updates fired within the same lane during the same tick are **batched** into one render. In React 18+ batching is *automatic everywhere* — not just inside React event handlers, but in promises, `setTimeout`, and native event callbacks too. Mechanically, multiple `setState` calls OR their (typically identical) lane onto the fiber; React schedules a single render for that lane and drains all queued updates in `updateQueue` in order. Updates of *different* lanes may be split across separate renders if their priorities differ — e.g. an urgent click update can render ahead of a pending transition.

## Entanglement

Sometimes two lanes that React would normally process separately **must** be processed together for correctness. React marks them **entangled** in the root's `entangledLanes`: whenever one is selected for a render, the entangled partners are pulled in too. The canonical cases:

- `useTransition` ensures a transition's updates stay consistent so you don't render a torn intermediate state across separate passes.
- `useDeferredValue` and certain Suspense retries entangle to avoid showing inconsistent UI.

Entanglement is the escape hatch that keeps the "just pick the highest lane" rule from producing visually inconsistent results.

```text
pending:  0b0000_0000_0010_0100   (DefaultLane | one TransitionLane)
pick highest set bit group → render those → clear them → repeat
entangled lanes get force-merged into the selected set
```

## Starvation Avoidance

A pure "always render the highest lane first" policy would let a busy stream of urgent updates **starve** a low-priority transition forever. React prevents this with **expiration times per lane**. When a lane first becomes pending, React stamps it with a deadline (urgent lanes expire almost immediately, transitions later, idle effectively never). On each scheduling pass, `markStarvedLanesAsExpired` checks the clock; any lane past its deadline is added to the set of lanes that must render *now*, synchronously, regardless of newer urgent work. So low-priority work is allowed to wait — but not indefinitely.

## Senior Pitfalls And Model

- **Lanes are about ordering, not threading.** Everything still runs on one thread; lanes decide *which updates land in which render pass*, letting an urgent update interrupt and finish before a transition resumes.
- **Transition lanes are a finite pool.** Many concurrent transitions cycle through the ~16 transition lanes; this is an implementation detail you rarely touch but explains why distinct transitions can occasionally share scheduling fate.
- **An interrupted render's lanes aren't lost.** If a higher lane preempts a transition mid-render, the transition's lanes remain pending on the root and are rescheduled — they don't get dropped, they get deferred.
- **Don't reason about exact bit values.** They've shifted across versions; reason in terms of *categories and relative urgency*, which is the stable contract.

## See Also

- [[work-loop-and-scheduling]]
- [[fiber-architecture]]
- [[transitions]]
- [[concurrent-rendering]]
- [[10-frameworks-and-stacks/react-native/performance/re-renders|RN Re-renders]]
