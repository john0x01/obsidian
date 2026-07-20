# Bit Manipulation

Bit manipulation operates directly on the binary representation of integers using bitwise operators. It solves problems where the data *is* a set of flags, where you need to squeeze many booleans into one word, or where an arithmetic identity has a far cheaper bitwise form — trading readability for speed, compactness, and the enumeration primitives behind bitmask dynamic programming.

## Binary Representation and Two's Complement

An unsigned integer is a positional base-2 number: bit i has weight 2^i. Signed integers use **two's complement**: the top bit is the sign, and negation is *invert all bits then add one* (`-x === ~x + 1`). This is the crucial identity — it means `x` and `-x` differ in a way that isolates the lowest set bit, and it means `~x === -x - 1`. There is a single representation of zero and arithmetic "just works" across the sign boundary, which is why hardware uses it.

## The Operators

- **AND `&`** — 1 only where both bits are 1. Used to *test* and *mask off* bits.
- **OR `|`** — 1 where either is 1. Used to *set* bits.
- **XOR `^`** — 1 where bits differ. Self-inverse (`a ^ a === 0`, `a ^ 0 === a`) and its own undo — the workhorse of bit tricks.
- **NOT `~`** — flips every bit (and, in two's complement, computes `-x - 1`).
- **Shifts** — `x << k` multiplies by 2^k; `x >> k` is an *arithmetic* right shift (sign-extending); `x >>> k` is a *logical* (unsigned) right shift, filling zeros.

## Masks: Set, Clear, Toggle, Test

Single-bit operations are the vocabulary of flag words. Let the mask be `1 << i`:

```ts
const set    = (x: number, i: number) => x |  (1 << i);   // force bit i to 1
const clear  = (x: number, i: number) => x & ~(1 << i);   // force bit i to 0
const toggle = (x: number, i: number) => x ^  (1 << i);   // flip bit i
const test   = (x: number, i: number) => (x >> i) & 1;    // read bit i → 0/1
```

A *field* mask `(1 << k) - 1` yields k low ones — handy for `x & ((1<<k)-1)` to keep the low k bits, which is exactly the power-of-two table-index trick in [[hash-functions]].

## Classic Tricks

- **Clear the lowest set bit:** `x & (x - 1)`. Subtracting 1 flips the lowest 1 and everything below it; ANDing keeps the rest. Iterating this runs once per set bit — **Kernighan's popcount** counts set bits in O(number of ones), not O(width).
- **Isolate the lowest set bit:** `x & -x` (relying on two's complement).
- **Power-of-two check:** `x > 0 && (x & (x - 1)) === 0` — a power of two has exactly one set bit, so clearing it yields 0.
- **XOR to find the unique element:** XOR every element of an array where all values appear twice except one; pairs cancel (`a ^ a = 0`), leaving the singleton. O(n) time, O(1) space, no hashing.

```ts
function popcount(x: number): number {
  let c = 0;
  while (x) { x &= x - 1; c++; }   // strip one set bit per iteration
  return c;
}
```

## Subset / Bitmask Enumeration

A bitmask represents a subset of up to ~30 elements as one integer — bit i means "element i present." This turns set-state into an array index, the core of **bitmask DP** (traveling-salesman, assignment, "cover all items" problems): `dp[mask]` is the best answer for the set `mask`, and transitions add one element with `mask | (1 << i)`.

```ts
for (let mask = 0; mask < (1 << n); mask++) { /* every subset of n items */ }

// Enumerate every SUBSET of a fixed mask, in O(number of submasks):
for (let sub = mask; sub > 0; sub = (sub - 1) & mask) { /* ... */ }
```

That submask trick — `(sub - 1) & mask` — walks exactly the subsets of `mask` and, summed over all masks, gives the elegant 3^n bound for subset-sum-style DP. See [[dynamic-programming]].

## Pitfalls in JS/TS

JavaScript numbers are IEEE-754 doubles, but **bitwise operators coerce to 32-bit signed integers**, operate, then convert back. Consequences a senior must internalize:

- Any value ≥ 2³¹ or with meaningful bits above bit 31 is **truncated / sign-flipped** by `&`, `|`, `^`, `<<`. `1 << 31` is *negative* (`-2147483648`). Never bit-twiddle values that need more than 31 usable bits — use `BigInt` (with its own `&`/`|`/`<<`) instead.
- `>>` is arithmetic (keeps sign); use `>>>` for a **logical** shift and to coerce to *unsigned* — `x >>> 0` is the idiom to read a value as uint32 (essential when handling hash outputs; see [[hash-functions]]).
- `<<` shift counts are taken **mod 32**, so `1 << 32 === 1`, silently — a nasty off-by-width bug.
- `~5` is `-6`, not `250` — `~` works on the full 32-bit two's-complement value, not a byte.

## Senior Pitfalls & Uses

Beyond the JS quirks: bitsets pack booleans at 1/8 the memory of a byte-array and let you AND/OR whole sets in one instruction (used in [[bloom-filters]] and database bitmap indexes). But undocumented bit tricks are unreadable — comment the intent. Prefer clarity unless the hot path or memory budget genuinely demands the bits.

## See Also
- [[hash-functions]] — masking, `>>> 0`, and bit mixing
- [[dynamic-programming]] — bitmask DP and subset enumeration
- [[bloom-filters]] — bit arrays as compact membership sets
- [[space-complexity]] — bitsets as a constant-factor memory win
