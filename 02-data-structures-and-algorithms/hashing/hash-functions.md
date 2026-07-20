# Hash Functions

A hash function maps an arbitrary key to a fixed-width integer used to pick a bucket in a [[hash-tables|hash table]]. Its whole job is to scatter keys uniformly and cheaply; the quality of that scatter is what turns the table's *average* O(1) into a reality rather than a hope.

## Desirable Properties

- **Determinism** — the same key must always produce the same hash. Non-negotiable; a key must land in the same bucket every time.
- **Uniformity** — outputs should spread evenly across the range, approximating the simple-uniform-hashing assumption. Clumping means long chains and O(n) lookups.
- **Avalanche** — flipping one input bit should flip about half the output bits. Without avalanche, similar keys (`user_1`, `user_2`) share high-order bits and collide under masking.
- **Speed** — the hash runs on *every* operation, so it must be far cheaper than the lookup it enables. A slow hash silently taxes the whole table.

Note the tension: a table hash wants uniformity + speed, not the *irreversibility* a cryptographic hash provides. Those are different goals (see below).

## Hashing Integers and Strings

For integers, the classic non-crypto mixer is **multiplicative (Fibonacci) hashing**: multiply by a constant and take the high bits.

```ts
// Knuth multiplicative hash: mix, then take the top `bits`.
const KNUTH = 2654435769; // ≈ 2^32 / φ
const hash = (x: number, bits: number) =>
  ((x * KNUTH) >>> (32 - bits)) >>> 0;   // high bits carry the most mixing
```

Multiplying spreads entropy from low bits into high bits; taking the *top* bits (not the bottom) is what gives avalanche.

For strings, the standard is **polynomial rolling hashing** — treat the string as digits of a base-B number, reduced mod a prime (or left to overflow in a fixed width):

```ts
function hashStr(s: string): number {
  let h = 0;
  for (let i = 0; i < s.length; i++) h = (h * 31 + s.charCodeAt(i)) | 0; // Java's choice
  return h >>> 0;
}
```

The *rolling* property matters beyond tables: because each character contributes a power of B, you can slide a window and update the hash in O(1) by subtracting the outgoing term and adding the incoming one — the engine behind Rabin–Karp substring search (see [[string-algorithms]]).

## Mapping the Hash to a Table: Modulo vs Bit-Masking

A wide hash must be folded into `[0, m)`.

- **Modulo** `h % m` works for any m and, with **prime** m, breaks up arithmetic patterns in weak hashes.
- **Bit-masking** `h & (m - 1)` works only when **m is a power of two**, and is dramatically faster (one AND vs a division). But masking keeps *only the low bits*, so it is unforgiving of hashes with weak low-bit mixing — precisely why multiplicative hashing takes the *high* bits, and why Java applies a supplemental `h ^= (h >>> 16)` spread before masking its power-of-two tables.

The rule: **power-of-two capacity + bit-mask** demands a well-avalanched hash; **prime capacity + modulo** is more forgiving of a mediocre one. Pick the pair together, never in isolation. See [[bit-manipulation]] for why `& (m-1)` equals `% m` when m is a power of two.

## Cryptographic vs Non-Cryptographic

- **Non-cryptographic** hashes (FNV-1a, MurmurHash, xxHash, CityHash) optimize for speed and distribution. They are the right default for hash tables — but they are *predictable*: given the algorithm, anyone can compute which keys collide.
- **Cryptographic** hashes (SHA-256, BLAKE3) add collision-resistance and pre-image resistance for security uses, but are far too slow for per-lookup table indexing. Different tool, different job — see [[08-quality-and-operations/security/hashing|Hashing (Security)]].

**SipHash** sits deliberately in between: a fast *keyed* pseudorandom function. It takes a secret 128-bit key chosen per process, so an attacker who knows the algorithm still cannot predict which keys collide. That defeats **hash-flooding DoS**, where crafted colliding keys collapse a table to O(n) per op. This is why Python (`dict`), Rust (`HashMap`), and Perl seed their string hashing with SipHash by default — the small per-hash cost buys immunity to adversarial input.

## Senior Pitfalls

- **A "fast" hash that ignores whole regions of the key** (e.g. hashing only the first 8 bytes of a long URL) is fast and catastrophically clustered. Uniformity must cover the *whole* key.
- **Combining sub-hashes with `+` or `^` alone** loses information and creates symmetric collisions (`(a,b)` vs `(b,a)`); use a multiply-and-mix combiner.
- **Reusing a table hash where you need integrity** (or vice versa) is a category error — table hashes have trivial collisions by design.
- The same distribution logic that a good hash provides is exactly what makes probabilistic structures like [[bloom-filters]] work; they lean on multiple independent, well-mixed hashes.

## See Also
- [[hash-tables]] — the structure this function drives
- [[collision-resolution]] — what happens when two keys hash equal
- [[bloom-filters]] — multiple hashes for probabilistic membership
- [[bit-manipulation]] — masking, shifts, and mixing bits
- [[string-algorithms]] — rolling hashes and Rabin–Karp
- [[08-quality-and-operations/security/hashing|Hashing (Security)]] — crypto hashes and SipHash
