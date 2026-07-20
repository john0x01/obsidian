# Greedy Algorithms

A greedy algorithm builds a solution by repeatedly making the choice that looks best **right now**, never reconsidering. It never backtracks and never revisits a decision — which makes greedy algorithms fast and simple, but correct only for a special class of problems. The hard part of "greedy" is never the code; it is *proving* the local choice leads to a global optimum.

## The Two Properties You Must Verify

Greedy is correct for a problem exactly when both of these hold:

1. **Greedy-choice property** — a globally optimal solution can be reached by making a locally optimal (greedy) choice at each step. Formally: there exists an optimal solution that *contains* the greedy first choice. This is what lets you commit without lookahead.
2. **Optimal substructure** — after making the greedy choice, what remains is a smaller instance of the same problem, and an optimal solution to the whole contains optimal solutions to the subproblem.

Optimal substructure is shared with [[dynamic-programming]]; the greedy-choice property is the *stronger* extra condition. DP considers all sub-solutions and picks the best; greedy gambles that one specific local choice is safe. When the gamble is provably safe, greedy wins because it explores exactly one path instead of a table of them.

## The Exchange Argument

The standard proof technique is the **exchange argument**. Take any optimal solution `O`; show that if `O` differs from the greedy solution `G` at the first point of divergence, you can *swap* greedy's choice into `O` without making it worse (and without violating feasibility). Repeat, transforming `O` into `G` one exchange at a time — since no step degrades the objective, `G` is at least as good as `O`, hence optimal.

Concretely, for **interval scheduling** (max non-overlapping intervals), greedy picks the interval with the *earliest finish time*. Exchange proof: if the optimal solution's first interval finishes later than greedy's, swap it for greedy's — greedy's finishes no later, so it can't conflict with anything the optimal solution scheduled afterward. Feasibility is preserved, count is unchanged, and we've moved one step toward the greedy solution.

## Matroid Intuition (Brief)

There is a deep theory here: greedy is *guaranteed* optimal precisely on **matroids** — set systems whose independent sets satisfy an exchange axiom (if `A`, `B` are independent and `|A| < |B|`, some element of `B` extends `A`). Kruskal's MST is the textbook case: forests form the *graphic matroid*, and greedily taking the cheapest edge that keeps the forest acyclic is optimal for any weights. When your problem's feasible sets form a matroid, greedy-by-weight is optimal — no separate proof needed. Weighted problems that *aren't* matroids (or the richer *matroid intersection*) generally need DP or flow.

## When Greedy Works

- **Interval scheduling / activity selection** — earliest finish time.
- **Huffman coding** — repeatedly merge the two lowest-frequency nodes; provably optimal prefix code.
- **Dijkstra's shortest paths** — greedily finalize the closest unfinalized vertex; correct because edge weights are non-negative (a shorter route can't emerge later). See [[shortest-paths]].
- **Kruskal / Prim MST** — cheapest safe edge; the cut property guarantees safety. See [[minimum-spanning-tree]].
- **Fractional knapsack** — take items by value/weight ratio, splitting the last one. Fractions make the greedy choice exchangeable.

## When Greedy Fails

- **0/1 knapsack.** You can't split items, so ratio-greedy breaks. Capacity `50`; items `(value 60, weight 10)`, `(100, 20)`, `(120, 30)`, with value/weight ratios `6, 5, 4`. Greedy by ratio takes the first two → value `160`, weight `30`, and the third (weight 30) no longer fits. Optimal is items two + three → value `220`, weight `50`. Greedy grabbed the high-ratio small item and stranded capacity it couldn't refill. **0/1 knapsack needs [[dynamic-programming]].**
- **Coin change with arbitrary denominations.** Greedy (take the largest coin ≤ remainder) is optimal for *canonical* systems like US coins, but not in general. Denominations `{1, 3, 4}`, target `6`: greedy takes `4 + 1 + 1 = 3 coins`; optimal is `3 + 3 = 2 coins`. Greedy's first choice (`4`) was locally best but globally wrong — the greedy-choice property fails, so you need DP.

## Senior Pitfalls

- **"It passed my test cases" is not a proof.** Greedy is the single most common source of *plausible-but-wrong* solutions. Always produce an exchange argument or a matroid, or fall back to DP.
- **Sort key correctness.** Interval scheduling by *earliest start* or *shortest duration* both look reasonable and are both wrong — only earliest *finish* works. The right greedy key is subtle.
- **Non-negativity in Dijkstra.** A single negative edge breaks the greedy-finalization invariant; use Bellman-Ford instead. This is a real bug, not a corner case.

## See Also

- [[dynamic-programming]]
- [[minimum-spanning-tree]]
- [[shortest-paths]]
- [[divide-and-conquer]]
- [[priority-queues]]
- [[union-find]]
