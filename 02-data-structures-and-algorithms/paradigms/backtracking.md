# Backtracking

Backtracking is a systematic way to search an **implicit decision tree**: you build a candidate solution one choice at a time, and the instant a partial candidate cannot possibly extend to a valid full solution, you abandon it ("prune") and back up to try the next option. It is depth-first search over a space of choices, and pruning is what separates it from hopeless brute force.

## The Choose → Explore → Un-Choose Loop

Every backtracking algorithm has the same shape: at each node you iterate over the available choices, and for each one you **choose** it (record it in the partial state), **explore** recursively, then **un-choose** it (undo the mutation) before trying the next. That symmetric mutate/undo around the recursive call is the defining move — it lets a single mutable state object represent every node of the tree without copying.

```ts
function backtrack(state: State, choices: Choice[], out: Solution[]) {
  if (isComplete(state)) { out.push(snapshot(state)); return; }
  for (const c of choices) {
    if (!isValid(state, c)) continue;   // prune: skip infeasible choices
    apply(state, c);                    // choose
    backtrack(state, nextChoices(c), out); // explore
    undo(state, c);                     // un-choose (restore state)
  }
}
```

The tree is *implicit* — it is never materialized; it exists only as the call stack. Depth equals the number of decisions (solution length); the branching factor is the number of choices per step. This is a DFS traversal (see [[graph-traversal]]) over a tree you generate on the fly.

## Pruning Is the Whole Game

Without pruning, backtracking is just an expensive enumeration of every leaf — pure brute force. Pruning cuts entire subtrees before descending into them, and it is the only reason backtracking is tractable on problems whose full space is astronomically large.

Two levels of pruning:

- **Constraint checking** — reject a choice that violates a rule *now* (`isValid` above). In N-Queens, before placing a queen you check its column and both diagonals against already-placed queens; an attacked square is skipped, killing that entire subtree.
- **Constraint propagation** — go further and deduce the *consequences* of a choice to prune future branches cheaply. In Sudoku, after placing a digit you remove it from the candidate sets of its row, column, and box; cells whose candidate set becomes empty prove the branch is dead before you even reach them. Choosing the **most-constrained variable** next (the empty cell with the fewest candidates) prunes far more than left-to-right order.

The art of backtracking is designing the *incremental* validity check so it is `O(1)` or cheap per step (maintain sets/bitmasks of occupied columns and diagonals rather than re-scanning the board), and ordering choices so failures surface early.

## Canonical Problems

- **N-Queens** — place `N` non-attacking queens; one queen per row, branch over columns, prune by column/diagonal [[bit-manipulation|bitmasks]].
- **Permutations / subsets / combinations** — the archetypes. Subsets branch on include/exclude each element (`2ⁿ` leaves); permutations branch over unused elements (`n!` leaves); combinations fix a start index to avoid reordering duplicates.
- **Sudoku** — fill cells subject to row/column/box constraints; constraint propagation is essential.
- **Word search / maze / path finding** — DFS on a grid marking visited cells on the way down and unmarking on the way back up (the un-choose step).
- **Graph coloring, Hamiltonian path, subset-sum** — classic NP-complete searches where backtracking with good pruning beats naive enumeration in practice.

## Complexity: Exponential Worst Case

Backtracking's worst case is the size of the search tree — `O(2ⁿ)` for subsets, `O(n!)` for permutations, `O(bᵈ)` in general for branching factor `b` and depth `d`. Pruning does **not** improve the worst-case bound (an adversarial input may prune nothing), but it can make the *typical* case dramatically smaller — this is why backtracking is the workhorse for NP-hard problems where no polynomial algorithm is known. Extra space is `O(d)` for the recursion stack plus the partial-solution state.

## Contrast with Brute Force and DP

Versus **brute force**: brute force generates every complete candidate then tests it; backtracking tests *incrementally* and abandons partial candidates early, so it explores a small pruned subtree instead of the full leaf set. Same worst case, wildly different practical cost.

Versus **[[dynamic-programming]]**: DP applies when subproblems *overlap* and you want a single optimal value or count — it memoizes and reuses. Backtracking applies when you must **enumerate distinct solutions** or when states don't overlap (each path is a different concrete assignment), so there's nothing to cache. If you find yourself backtracking and hitting the *same subproblem state* repeatedly, add memoization and you've turned it into DP (this is exactly the bridge for problems like word-break or regex matching).

## Senior Pitfalls

- **Forgetting to un-choose** — the single most common bug; state leaks across branches and corrupts every subsequent path. The `undo` must exactly mirror `apply`.
- **Copying state at every node** — turns an `O(d)` space algorithm into `O(d · statesize)` and adds a large constant; mutate-and-undo instead, and only `snapshot` at accepted leaves.
- **Weak pruning** — a backtracker that "works" on tiny inputs but hangs on real ones almost always has a missing or late constraint check; prune as early and as cheaply as possible, and order choices most-constrained-first.
- **Duplicate solutions** — without a canonical ordering (fixed start index, skipping equal siblings) you enumerate the same set/permutation many times.

## See Also

- [[recursion]]
- [[graph-traversal]]
- [[bit-manipulation]]
- [[dynamic-programming]]
- [[divide-and-conquer]]
- [[tree-traversals]]
