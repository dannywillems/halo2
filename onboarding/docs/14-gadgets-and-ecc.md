---
sidebar_position: 14
title: halo2_gadgets and the ECC Chip
description: The chip / instruction trait pattern, and a tour of the elliptic-curve gadget that exemplifies it.
---

# halo2_gadgets and the ECC Chip

## 1. Why this chapter exists

`halo2_proofs` gives you the proving system. `halo2_gadgets`
gives you a library of common circuits already written for you.
The library has two architectural patterns worth knowing: the
chip / instruction-trait split, and the convention that
gadgets are written against a trait so multiple chips can satisfy
them. The ECC gadget is the canonical example. If you ever write
a chip of your own, copy its shape.

## 2. Definitions

**Definition (Chip).** A type that owns a `Config` (a set of
columns and gates) and provides typed methods that synthesize
assignments using those columns. Defined in chapter 06.

**Definition (Instruction trait).** A trait whose methods name
the operations a chip should provide (e.g. `add`, `mul`,
`witness_point`). The gadget code uses only the trait; the
trait can be implemented by multiple chips with different
column / gate trade-offs.

**Definition (Fixed-base scalar multiplication).** Computing
$[k]G$ for a *fixed* base point $G$ known at circuit definition
time. The chip can precompute a windowed lookup table and
constrain the multiplication efficiently.

**Definition (Variable-base scalar multiplication).** Computing
$[k]P$ for a witnessed point $P$. Cannot use precomputed tables;
relies on the incomplete-and-then-complete double-and-add gates.

## 3. The code

### 3.1 The crate surface

**Source:** [`halo2_gadgets/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/lib.rs#L1-L35)

Four top-level modules:

- `ecc`: Pallas / Vesta points.
- `poseidon`: Poseidon sponge.
- `sinsemilla`: Sinsemilla hash + Merkle path.
- `sha256`: SHA-256 (feature-gated).
- `utilities`: shared building blocks (`cond_swap`,
  `decompose_running_sum`, `lookup_range_check`).

### 3.2 The `EccInstructions` trait

**Source:** [`halo2_gadgets/src/ecc.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc.rs#L14-L62)

A few associated types make the trait surface clean:

- `Point`, `NonIdentityPoint`, `X`: witness representations of
  a curve point and its $x$-coordinate.
- `ScalarVar`, `ScalarFixed`, `ScalarFixedShort`: three scalar
  widths. The "short" scalar covers values fitting in 64 signed
  bits and exists so that fixed-base scalar mul for small
  scalars can be cheaper.
- `FixedPoints`: an enumeration of all fixed bases the chip
  supports.

Methods include `witness_point`, `add`, `add_incomplete`,
`mul`, `mul_fixed`, `mul_fixed_short`, `mul_fixed_base_field_elem`,
plus a `constrain_equal` for cross-region equality. Each method
is split into one or more rows of a column-dense gate; see the
chip impl.

### 3.3 The `EccChip` config

**Source:** [`halo2_gadgets/src/ecc/chip.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip.rs#L130-L170)

The config records which advice columns and selectors are
allocated for each sub-gate. The columns are *shared* across
sub-gates; each selector turns on a different polynomial
identity. This is what makes the chip dense.

### 3.4 The `EccChip` itself

**Source:** [`halo2_gadgets/src/ecc/chip.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip.rs#L225-L270)

The chip stores its `Config` and the parameter type that fixes
the set of base points. `EccChip::configure` is where the
sub-gates are declared.

### 3.5 The sub-gates

Each sub-gate lives in its own file:

- [`chip/witness_point.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/witness_point.rs):
  enforce the curve equation $y^2 = x^3 + ax + b$ on a witnessed
  $(x, y)$.
- [`chip/add_incomplete.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/add_incomplete.rs):
  incomplete addition, valid when $P \neq Q$ and neither is the
  identity. Cheaper than the complete formula.
- [`chip/add.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/add.rs):
  complete addition. Used wherever the inputs are not provably
  distinct.
- [`chip/mul.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/mul.rs):
  variable-base scalar mul (double-and-add).
- [`chip/mul_fixed/`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/mul_fixed):
  fixed-base scalar mul with precomputed windowed tables.

### 3.6 The shared utilities

**Source:** [`halo2_gadgets/src/utilities/lookup_range_check.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/utilities/lookup_range_check.rs#L1-L60)

`LookupRangeCheck` is used by ECC and Sinsemilla alike to assert
that a witnessed value fits in $K$ bits, using a lookup table of
size $2^K$. It is one of the most-touched files in the repo (see
hot files in chapter 02); changes here affect many downstream
chips.

## 4. Failure modes

- **Using `add_incomplete` for inputs that may collide.** If
  $P = Q$ or one of them is the identity, the incomplete formula
  produces a wrong answer with no failure indication; the
  resulting constraint will hold on a wrong witness. Always use
  `add` (complete) unless you have a separate argument for
  distinctness.
- **Forgetting `EccChip::config().fixed_bases`.** Fixed-base
  scalar mul needs the chip to be configured with the specific
  base point. Adding a new fixed base requires regenerating the
  precomputed windowed table.
- **Mixing scalar types.** `ScalarVar`, `ScalarFixed`, and
  `ScalarFixedShort` are not interchangeable. Passing a
  `ScalarFixedShort` to a method that expects a `ScalarFixed`
  is a compile-time error; passing the wrong-width witness to
  either of them is a runtime constraint failure.
- **Skipping range checks on witness scalars.** The ECC chip
  assumes scalars are properly range-checked elsewhere
  (typically via `LookupRangeCheck`). Forgetting to range-check
  a witness scalar leaves the chip open to canonicalization
  attacks (e.g. scalars exceeding the curve order).

## 5. Spec pointers

- The
  [Halo 2 Book "Elliptic curve cryptography" chapter](https://zcash.github.io/halo2/design/gadgets/ecc.html)
  derives the in-circuit addition formulas. Section
  [Incomplete and complete addition](https://zcash.github.io/halo2/design/gadgets/ecc/addition.html)
  is the prerequisite reading for `chip/add*.rs`.
- [Zcash Protocol Spec, section 5.4.9 "Pallas and Vesta"](https://zips.z.cash/protocol/protocol.pdf):
  the curve constants and group structure.
- [Hopwood et al., "Recursive Proof Composition without a Trusted
  Setup"](https://eprint.iacr.org/2019/1021): the recursion
  framing that motivated the ECC chip's design.

## 6. Exercises

1. Open
   [`halo2_gadgets/src/ecc.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc.rs)
   and identify the difference between `Point` and
   `NonIdentityPoint`. Why does the gadget distinguish them?
2. Read
   [`halo2_gadgets/src/ecc/chip/add_incomplete.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/add_incomplete.rs)
   and identify the gate degree. Compare with `add.rs`'s
   complete-addition gate degree; the increase explains why the
   incomplete variant exists.
3. Run the ECC tests:

   ```bash
   cargo test --release -p halo2_gadgets ecc
   ```

   Identify each integration test and the chip API it exercises.
4. Pick one fixed-base scalar mul test, find the chosen base in
   the test setup, and confirm the precomputed table for that
   base lives in
   [`halo2_gadgets/src/ecc/chip/constants.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/constants.rs).

### Answers in the code

- Exercise 1: `Point` includes the identity (as the special
  encoding $(0, 0)$); `NonIdentityPoint` is provably not the
  identity. The split lets the chip pick incomplete addition
  whenever both arguments are `NonIdentityPoint`.
- Exercise 2: the incomplete gate has degree 3; the complete
  gate has degree 6 in halo2's encoding. The 2x degree increase
  doubles the extended-domain size and roughly doubles prover
  cost.

## 7. Further reading

- The
  [Halo 2 Book "Variable-base scalar multiplication" chapter](https://zcash.github.io/halo2/design/gadgets/ecc/var-base-scalar-mul.html)
  for the full double-and-add walkthrough.
