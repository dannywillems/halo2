---
sidebar_position: 15
title: Hash Gadgets -  Poseidon, Sinsemilla, SHA-256
description: Three hash gadgets each chosen for a specific reason, and how to read their chip implementations.
---

# Hash Gadgets: Poseidon, Sinsemilla, SHA-256

## 1. Why this chapter exists

The three hash chips in `halo2_gadgets` exist because no single
hash function dominates in the in-circuit setting. Poseidon is
the cheapest algebraic hash; Sinsemilla is purpose-built for
Pallas-curve commitments and used in the Orchard Merkle tree;
SHA-256 is required by Bitcoin / Zcash interop and uses the
lookup-heavy table16 chip. Each one is a case study in a
different chip architecture. If you ever add a new hash chip,
the patterns here are the templates to copy.

## 2. Definitions

**Definition (Algebraic hash).** A hash whose round function is
a low-degree polynomial in the field. Cheap inside a circuit
because each round becomes one (or a few) gates. Poseidon is the
canonical example.

**Definition (Sponge construction).** A hash mode that absorbs
input in `RATE`-sized chunks and squeezes output in similar
chunks. Halo 2's Poseidon API exposes the sponge as `Sponge<F,
S, T, RATE>`, where `T` is the state width and `RATE = T - 1`
for the standard variant.

**Definition (Sinsemilla).** A SNARK-friendly hash family
designed by Bowe and Hopwood, optimized for the Pallas curve.
The hash absorbs a message as a sequence of $K$-bit chunks and
accumulates the running point $P$ as
$P \leftarrow P + Q_i$ where $Q_i$ is a precomputed generator
indexed by the chunk. The construction is in
[ZIP 224](https://zips.z.cash/zip-0224).

**Definition (Merkle path verification).** Given a root $r$, a
leaf value $\ell$, and a sibling list, the gadget computes
`root(path, leaf)` and constrains it equal to $r$. Halo 2's
implementation in
[`halo2_gadgets::sinsemilla::merkle`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs)
uses Sinsemilla as the underlying hash, hence MerkleCRH.

**Definition (SHA-256 table16 chip).** An in-circuit
implementation of SHA-256 that uses a 16-bit "spread table" to
turn bitwise XOR / AND operations into lookups, dramatically
reducing the per-bit constraint cost.

## 3. The code

### 3.1 The `Spec` trait (halo2_poseidon)

```rust file=../../halo2_poseidon/src/lib.rs#L35-L120 title="halo2_poseidon/src/lib.rs"
```
A `Spec<F, T, RATE>` implementation provides:

- `full_rounds()`, `partial_rounds()`: round counts.
- `sbox(val) -> F`: the S-box.
- `secure_mds()`: choice of MDS matrix.
- `constants()`: the round constants.

The crate ships
[`P128Pow5T3`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_poseidon/src/p128pow5t3.rs)
which is the parameters Orchard uses.

### 3.2 The `Pow5Chip` (in-circuit Poseidon)

```rust file=../../halo2_gadgets/src/poseidon/pow5.rs#L35-L100 title="halo2_gadgets/src/poseidon/pow5.rs"
```
The chip implements `PoseidonInstructions` using the standard
$x^5$ S-box and the round constants from
[`halo2_poseidon::p128pow5t3`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_poseidon/src/p128pow5t3.rs).
It allocates `WIDTH + 1` advice columns (one per state element
plus one for the result of the S-box of the first element); the
remaining round operations are folded into compound gates.

### 3.3 The Poseidon sponge and hash API

```rust file=../../halo2_gadgets/src/poseidon.rs#L115-L170 title="halo2_gadgets/src/poseidon.rs"
```
`Sponge<F, S, T, RATE>` is the in-circuit sponge; `Hash<F, S, T,
RATE, L>` is the convenience type that fixes an absorption mode.
The `L` parameter records the length of a single hash invocation
at compile time, which the type system uses to ensure correct
padding.

### 3.4 Sinsemilla: the chip

```rust file=../../halo2_gadgets/src/sinsemilla/chip.rs#L100-L160 title="halo2_gadgets/src/sinsemilla/chip.rs"
```
`SinsemillaChip<Hash, Commit, Fixed, Lookup>` is generic over
four type parameters because Sinsemilla is used in two flavours
(a hash and a commitment) over two domains (MerkleCRH and
SinsemillaCommit), each with its own generator table. The hot
file
[`chip/hash_to_point.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs)
contains the inner loop that turns a chunk index into a
generator lookup and accumulates the running point. This is the
file with the most changes per year in the entire `halo2_gadgets`
crate.

### 3.5 Sinsemilla-Merkle

```rust file=../../halo2_gadgets/src/sinsemilla/merkle.rs#L18-L75 title="halo2_gadgets/src/sinsemilla/merkle.rs"
```
`MerkleInstructions` defines the per-layer hash, and
`MerklePath<...>::calculate_root` runs the path verification.
The level-specific personalization (each Merkle layer absorbs the
layer number to prevent attacks across heights) is in
[`hash_to_point.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs).

### 3.6 SHA-256 table16 chip

```rust file=../../halo2_gadgets/src/sha256/table16.rs#L230-L290 title="halo2_gadgets/src/sha256/table16.rs"
```
The `Table16Chip` decomposes each 32-bit SHA-256 word into two
16-bit halves and uses a 16-bit "spread" lookup
([`table16/spread_table.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sha256/table16/spread_table.rs))
to turn bitwise XOR / AND into constant-cost lookups. The chip
is feature-gated by `unstable-sha256-gadget`; if you do not
enable that feature, `halo2_gadgets::sha256` is empty.

The cost trade-off: a single SHA-256 block costs roughly the
order of $2^{16}$ table rows (mostly shared across all SHA-256
invocations in the circuit), plus a few thousand gate rows per
block.

### 3.7 Where this fits in Orchard

Each hash family is chosen for a specific Orchard primitive. See
[the protocol-context chapter](./protocol-context) for the
underlying definitions; the mapping is:

| Orchard primitive ([spec](https://zips.z.cash/protocol/protocol.pdf))                            | Hash family             | Chip                                                              |
| ------------------------------------------------------------------------------------------------ | ----------------------- | ----------------------------------------------------------------- |
| `NoteCommit^Orchard` (the note commitment)                                                       | `SinsemillaCommit`       | `halo2_gadgets::sinsemilla::CommitDomain`                          |
| `MerkleCRH^Orchard` (per-layer Merkle hash)                                                      | Sinsemilla hash         | `halo2_gadgets::sinsemilla::merkle::MerklePath`                    |
| `Commit^ivk` (incoming viewing key derivation)                                                   | `SinsemillaCommit`       | `halo2_gadgets::sinsemilla::CommitDomain`                          |
| `PRF^nfOrchard(nk, rho)` (nullifier scalar)                                                      | Poseidon                | `halo2_gadgets::poseidon::Hash` over `Pow5Chip`                    |
| Note encryption key derivation `KDF^Orchard`                                                     | Blake2b (out-of-circuit) | (no in-circuit chip; runs in the wallet)                           |
| Sapling-pool / Bitcoin / Zcash interop hashing                                                   | SHA-256                 | `halo2_gadgets::sha256::Table16Chip` (feature `unstable-sha256-gadget`) |

This is why Sinsemilla and Poseidon dominate this chapter: every
*in-circuit* Orchard hash is one of those two. SHA-256 is in the
crate for legacy and interop reasons (e.g. proving statements
about transparent-pool addresses), but Orchard itself does not
use it on the proving side.

The Merkle-layer personalization that
`hash_to_point.rs` enforces is what binds a leaf to its layer
number; without it, a forged `MerklePath` could move a commitment
up or down the tree. The Orchard tree is fixed-depth
`MERKLE_DEPTH_ORCHARD = 32`; the same constant appears verbatim
in [`sinsemilla/merkle.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs#L210)
and in the Zcash Protocol Specification's "MerkleCRH^Orchard"
section.

The Poseidon `Spec` that Orchard actually uses is
`P128Pow5T3` (8 full rounds, 56 partial rounds, x^5 S-box),
imported in the chip's tests under the alias `OrchardNullifier`:

```rust file=../../halo2_gadgets/src/poseidon/pow5.rs#L613-L617 title="halo2_gadgets/src/poseidon/pow5.rs"
```

That alias is the most explicit Orchard reference in the crate;
treat it as a flag that whatever the test is exercising will
appear in the deployed Orchard circuit.

## 4. Failure modes

- **Mismatched `Spec`.** The in-circuit `Pow5Chip` and the
  out-of-circuit `halo2_poseidon::Hash` must use the *same*
  `Spec`. Two Specs with different round constants produce two
  different hash functions; the circuit will be unprovable.
- **Sinsemilla generator drift.** The generator table inside
  `SinsemillaChip` must match the spec's fixed bases. Adding a
  new generator without updating both sides silently changes
  the hash.
- **Forgetting the Merkle layer personalization.** A naive
  Merkle implementation that hashes only `(left || right)` is
  vulnerable to second-preimage attacks across tree heights.
  Sinsemilla-Merkle absorbs the layer number; do not skip this
  if you write a new Merkle gadget.
- **Compiling SHA-256 without the feature.** A
  `use halo2_gadgets::sha256;` will fail to link unless the
  feature flag is enabled.

## 5. Spec pointers

- [Poseidon paper, eprint 2019/458](https://eprint.iacr.org/2019/458):
  the original Poseidon design.
- [The Halo 2 Book "Sinsemilla" chapter](https://zcash.github.io/halo2/design/gadgets/sinsemilla.html)
  derives the chip's incomplete-addition trick.
- [ZIP 224 "Sinsemilla Hash Function"](https://zips.z.cash/zip-0224):
  the normative spec the chip targets.
- [FIPS 180-4, "Secure Hash Standard"](https://nvlpubs.nist.gov/nistpubs/FIPS/NIST.FIPS.180-4.pdf):
  SHA-256.
- [The Halo 2 Book "SHA-256" chapter](https://zcash.github.io/halo2/design/gadgets/sha256.html)
  for the 16-bit table chip design.

## 6. Exercises

1. Open
   [`halo2_poseidon/src/p128pow5t3.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_poseidon/src/p128pow5t3.rs)
   and identify the round constants and MDS matrix. Note the
   `full_rounds()` and `partial_rounds()` counts; cross-check
   them against the Poseidon paper.
2. Run the Poseidon benchmark:

   ```bash
   cargo bench -p halo2_gadgets --bench poseidon
   ```

   Identify the dominant cost: gates, lookups, or MSM.
3. Trace one absorption step inside
   [`halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs).
   Identify the lookup that picks the generator, and the
   accumulator update.
4. Build the SHA-256 benchmark with the feature flag:

   ```bash
   cargo bench -p halo2_gadgets --bench sha256 \
       --features unstable-sha256-gadget
   ```

   Compare per-block cost against a single Poseidon
   permutation.

### Answers in the code

- Exercise 1: `P128Pow5T3` has 8 full rounds and 56 partial
  rounds (for the standard 128-bit security parameters).
- Exercise 3: the lookup is into the per-domain generator table
  declared in
  [`sinsemilla/chip/generator_table.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/chip/generator_table.rs);
  the accumulator update uses incomplete addition.

## 7. Further reading

- The
  [Halo 2 Book "16-bit table chip"](https://zcash.github.io/halo2/design/gadgets/sha256/table16.html)
  for the spread-table technique.
- [Pasta Sinsemilla generator derivation](https://github.com/zcash/zcash/issues/4502)
  for the curve generator selection process.
