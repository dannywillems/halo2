---
sidebar_position: 1
title: Crate and Module Map
description: A tour of the halo2 workspace, what each crate owns, and where to look first.
---

# Crate and Module Map

## 1. Why this chapter exists

Before reading anything else, you need a map. The halo2 repository
is a four-crate Cargo workspace, but the names overlap with each
other and with the public Rust ecosystem (the
[`halo2`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2/Cargo.toml)
crate is a thin shim; the real proving system lives in
[`halo2_proofs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml)).
Most newcomers waste time searching the wrong crate. By the end of
this chapter you will know which module owns each of: the
arithmetization, the prover, the verifier, the polynomial
commitment, the chips, and the dev tooling.

## 2. Definitions

We use the terminology that the halo2 codebase uses internally.

**Definition (Workspace).** A Cargo workspace is a set of crates
sharing a single `Cargo.lock`. The halo2 workspace is declared in
[`Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/Cargo.toml)
with members `halo2`, `halo2_gadgets`, `halo2_poseidon`,
`halo2_proofs`.

**Definition (Crate).** Each member is a Cargo crate with its own
`Cargo.toml`, its own version, and its own crates.io release
schedule.

**Definition (PLONKish circuit).** A circuit expressed as a
rectangular table of cells with $n = 2^k$ rows and a fixed number
of columns, plus a set of low-degree polynomial constraints and
copy constraints over those cells. The arithmetization is defined
formally in
[`plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs);
chapter 05 covers it in detail.

**Definition (Chip vs Gadget).** A *chip* is a Rust type that
allocates columns and gates on a `ConstraintSystem` and exposes
typed assignment methods. A *gadget* is a thin layer that calls one
or more chips through a trait, without itself owning columns. The
distinction is made precise in
[`halo2_gadgets/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/lib.rs).

## 3. The code

### 3.1 The workspace manifest

The workspace lists four crates and nothing else:

**Source:** [`Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/Cargo.toml#L1-L7)

### 3.2 The `halo2` shim crate

The crate named `halo2` is a re-export shim. Its only dependency is
`halo2_proofs`, and its `lib.rs` re-exports the public surface so
users can write `halo2::plonk::create_proof` instead of
`halo2_proofs::plonk::create_proof`. Look here only if you want to
understand the public-version-vs-internal-version dance; do not put
code here.

**Source:** [`halo2/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2/Cargo.toml#L1-L24)

### 3.3 `halo2_proofs`: the proving system

This is the crate every other crate (and every downstream user)
depends on. Its public modules:

**Source:** [`halo2_proofs/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/lib.rs)

What each module owns:

- `arithmetic`: FFT, NTT, multi-scalar multiplication (`best_multiexp`),
  parallel helpers, and re-exports from `pasta_curves::arithmetic`.
  Covered in chapter 03.
- `circuit`: `Chip`, `Cell`, `AssignedCell`, `Layouter`,
  `SimpleFloorPlanner`, `TableLayouter`, `Value<F>`. Covered in
  chapter 06.
- `plonk`: the arithmetization (`Circuit`, `ConstraintSystem`,
  `Column`, `Selector`, `Expression`), keygen
  (`keygen_vk`, `keygen_pk`), `create_proof`, `verify_proof`, and
  the lookup / permutation / vanishing arguments. Covered in chapters
  05, 07, 08, and 11.
- `poly`: polynomial domains (`EvaluationDomain`), commitments
  (`commitment::Params`), and the multiopen wrapper. Covered in
  chapters 04, 09, and 10.
- `transcript`: Fiat-Shamir transcripts. Covered in chapter 12.
- `dev`: `MockProver`, `CircuitCost`, `CircuitGates`,
  `TracingFloorPlanner`, `CircuitLayout`. Covered in chapter 13.

### 3.4 `halo2_gadgets`: reusable chips

**Source:** [`halo2_gadgets/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/lib.rs)

The top-level modules are intentionally narrow:

- `ecc`: Pallas / Vesta point operations as a chip and a gadget
  trait `EccInstructions`. Chapter 14.
- `poseidon`: the in-circuit Poseidon sponge built on top of the
  field-only `halo2_poseidon` crate. Chapter 15.
- `sinsemilla`: the Sinsemilla hash and the Merkle gadget that
  consumes it. Chapter 15.
- `sha256`: only compiled with `--features unstable-sha256-gadget`.
  Chapter 15.
- `utilities`: `cond_swap`, `decompose_running_sum`,
  `lookup_range_check`. Frequently reused building blocks.

### 3.5 `halo2_poseidon`: a field-only specification

The Poseidon round constants, the MDS matrix, and the permutation
function in plain field arithmetic. No `halo2_proofs` dependency.
The in-circuit chip in `halo2_gadgets::poseidon::pow5` uses this
crate as the spec it must match.

**Source:** [`halo2_poseidon/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_poseidon/Cargo.toml)

## 4. Failure modes

- **Edits in the wrong crate.** Adding a new chip to `halo2_proofs`
  rather than `halo2_gadgets` is the single most common newcomer
  mistake. The rule: anything that fixes columns, defines a gate,
  or instantiates a chip belongs in `halo2_gadgets`. Only the
  arithmetization itself, the prover, the verifier, the dev tooling,
  and the polynomial commitment scheme belong in `halo2_proofs`.
- **Forgetting the shim.** Downstream users sometimes depend on
  `halo2` and `halo2_proofs` simultaneously, ending up with two
  copies of every type. Pick one.
- **Forgetting that `halo2_poseidon` is not a chip.** It is a
  spec-only crate. Importing it does not give you a circuit; you
  also need `halo2_gadgets::poseidon`.

## 5. Spec pointers

- The
  [Halo 2 Book "Concepts" chapter](https://zcash.github.io/halo2/concepts.html)
  describes the same workspace from a user's perspective; it is
  the canonical companion to this chapter.
- The PLONK paper
  ([eprint 2019/953](https://eprint.iacr.org/2019/953))
  introduces the arithmetization that `halo2_proofs::plonk`
  implements; section 4 (the standard PLONK arithmetization) is
  the most directly relevant.

## 6. Exercises

1. Open
   [`halo2_proofs/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/lib.rs)
   and list each `pub mod`. For each one, write a single sentence
   saying which chapter of this course covers it. Compare against
   section 3.3 above.
2. Find the line in
   [`halo2_proofs/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/lib.rs)
   that re-exports `pasta_curves` under the name `pasta`. Note that
   the same trick lets downstream code stay decoupled from the curve
   crate version.
3. Modify the file `halo2_proofs/examples/simple-example.rs` to
   print the workspace version of `halo2_proofs` at runtime using
   `env!("CARGO_PKG_VERSION")`. Run
   `cargo run --example simple-example` and confirm the printed
   version matches the value in
   [`halo2_proofs/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml).

### Answers in the code

- Exercise 1: the public modules are listed in
  [`halo2_proofs/src/lib.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/lib.rs).
- Exercise 2: the line is `pub use pasta_curves as pasta;` near the
  top of the same file.
- Exercise 3: the version is `0.3.2` in
  [`halo2_proofs/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml).

## 7. Further reading

- [docs.rs/halo2_proofs](https://docs.rs/halo2_proofs/0.3.2/halo2_proofs/)
  for the rendered API.
- [docs.rs/halo2_gadgets](https://docs.rs/halo2_gadgets/0.4.0/halo2_gadgets/)
  for the gadget catalogue.
