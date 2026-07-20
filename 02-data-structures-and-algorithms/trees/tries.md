# Tries

A **trie** (prefix tree) is a search tree keyed by the *sequence* of symbols in a key rather than by comparing whole keys. The path from the root to a node spells a prefix, so all keys sharing a prefix share a subtree. It solves problems that hashing cannot: prefix queries, ordered iteration, and longest-prefix matching — all in time proportional to key length, independent of how many keys are stored.

## The Idea and Invariant

Each node represents one prefix. A node holds a **map from next symbol to child** plus a **terminal flag** marking whether the prefix is itself a stored key (and optionally an associated value).

```ts
class TrieNode {
  children = new Map<string, TrieNode>();
  isEnd = false;
}
```

The invariant: the concatenation of edge labels along any root-to-node path equals the prefix that node represents; a key exists iff its path ends at a node with `isEnd = true`.

```ts
function insert(root: TrieNode, key: string) {
  let n = root;
  for (const ch of key) {
    if (!n.children.has(ch)) n.children.set(ch, new TrieNode());
    n = n.children.get(ch)!;
  }
  n.isEnd = true;
}
```

Search is the same walk without creating nodes; a **prefix query** ("is any key prefixed by P?") is just the walk for P — if it doesn't fall off the tree, the subtree under the landing node holds every completion.

## Complexity

Let L be the key length and Σ the alphabet size.

- **insert / search / prefix-walk:** **O(L)** — one step per symbol. Crucially this is **independent of n**, the number of keys. Compare a [[binary-search-trees|balanced BST]] of strings at `O(L·log n)` (log n comparisons, each up to L characters) or sorting.
- **enumerate all keys under a prefix:** O(size of that subtree), and they come out in **sorted order** via DFS.
- **space:** the pain point. A node per distinct prefix, and if children are stored as a fixed array of size Σ (fast, `O(1)` branching), each node costs O(Σ) even when nearly empty — worst case O(n·L·Σ). A hash/`Map` for children trades a little speed for far less memory on sparse alphabets.

## Variants and Trade-offs

- **Compressed trie / radix tree (Patricia):** collapse every chain of single-child nodes into one edge labeled with a *substring*. This removes the long thin tails that dominate a plain trie's node count, cutting memory dramatically and shortening lookups. A **Patricia trie** takes this to the bit level (binary alphabet, branch on the next differing bit) — the classic structure behind IP routing tables.
- **Ternary search trie:** each node stores one character with lo/eq/hi children — a middle ground that saves memory versus array-backed tries while keeping fast lookups.

## Trie vs Hashing

A [[hash-maps-and-sets|hash map]] gives O(1) *average* membership but destroys order: no prefix queries, no range/sorted iteration, and a worst case that degrades under adversarial or colliding keys (see [[collision-resolution]]). A trie instead offers:

- **Prefix queries and autocomplete** for free.
- **Ordered / lexicographic traversal** for free.
- **Longest-prefix match**, which hashing cannot express.
- A **worst case of O(L)** with no dependence on a hash function — no [[08-quality-and-operations/security/hashing|hash-flooding DoS]] surface.

The costs: higher memory and worse locality (pointer chasing per symbol) than a compact hash table. When you only need membership or exact-key lookup, hashing usually wins.

## When to Use It

- **Autocomplete / typeahead** and spell-checkers — the defining use case.
- **Dictionaries and word games** where you must test many prefixes.
- **IP longest-prefix routing** — forwarding tables are radix/Patricia tries so a router can match the most specific matching subnet; see [[04-databases/search-engines|Search Engines]] for related indexing ideas.
- **Multi-pattern string matching** — a trie of patterns is the skeleton of Aho–Corasick (see [[string-algorithms]]).

Avoid tries when keys are long and non-prefix-sharing (little sharing means little savings), or when memory is tight and you only need exact lookup.

## Senior Pitfalls

- **Deletion** must prune now-empty, non-terminal nodes upward, or the tree leaks; but stop pruning as soon as a node is still terminal or has other children.
- **Alphabet assumptions:** an array-of-26 works for lowercase ASCII but breaks on Unicode — decide whether you key by code unit, code point, or byte. UTF-8 byte tries are common and language-agnostic.
- **Locality tax:** the O(L) bound hides many cache misses; a benchmark against a hash map can surprise you — see [[big-o-vs-reality]].
- Don't confuse a trie with a **suffix tree/automaton**; those index all *suffixes* of one text for substring search, a different problem.

## See Also
- [[string-algorithms]] — pattern matching and suffix structures
- [[hash-maps-and-sets]] — the exact-lookup alternative
- [[04-databases/search-engines|Search Engines]] — prefix and term indexing at scale
- [[bit-manipulation]] — Patricia tries branch on bits
- [[binary-search-trees]] — comparison-based ordered maps
