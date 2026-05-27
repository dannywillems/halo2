---
sidebar_position: 11
title: Prover and Verifier Flow
description: How keygen, create_proof, and verify_proof stitch the arguments and commitments together.
---

# Prover and Verifier Flow

## 1. Why this chapter exists

Chapters 03-10 cover the arguments and commitments individually.
This chapter is the integration test: it walks through what
`keygen_vk`, `keygen_pk`, `create_proof`, and `verify_proof`
actually do, in order, and where each previous chapter's
machinery slots in. If you debug a "verifier returned OK but the
proof is wrong" issue, this is the chapter you re-read.

## 2. Definitions

**Definition (Verifying key, VK).** A bundle containing the
constraint system shape, the evaluation domain, the commitments
to the fixed columns, and the commitments to the permutation
polynomials. The VK is what circumvents the need to recommunicate
the circuit shape per proof. Defined in
[`halo2_proofs/src/plonk.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk.rs).

**Definition (Proving key, PK).** A `VerifyingKey` plus the fixed
columns themselves and the permutation polynomials in coefficient
form. The PK contains everything the prover needs to recompute
the FFTs; the VK contains everything the verifier needs to check
openings.

**Definition (Proof).** A byte string consumed by
`TranscriptRead`. Structurally it is the concatenation of: advice
column commitments (one per circuit, per advice column),
permutation grand-product commitments, lookup commitments,
vanishing polynomial commitments, multiopen commitments, and a
final IPA opening.

## 3. The code

### 3.1 Key generation

```rust file=../../halo2_proofs/src/plonk/keygen.rs#L189-L244 title="halo2_proofs/src/plonk/keygen.rs"
```
`keygen_vk` walks the circuit, builds the `ConstraintSystem`,
synthesizes the fixed columns and selectors using a witness-free
copy of the circuit, runs selector combining, computes the
permutation polynomials, and commits to everything.

```rust file=../../halo2_proofs/src/plonk/keygen.rs#L246-L337 title="halo2_proofs/src/plonk/keygen.rs"
```
`keygen_pk` takes the `VerifyingKey` and re-runs the same
synthesis to fill in the L1 / extended-domain caches used by the
prover.

### 3.2 The prover

```rust file=../../halo2_proofs/src/plonk/prover.rs#L33-L100 title="halo2_proofs/src/plonk/prover.rs"
```
The full prover flow (file is 786 lines; reading it once is
worth a study session by itself):

1. Hash the verifying key into the transcript (binds the proof to
   a particular circuit shape).
2. For each circuit (when batching multiple circuits), synthesize
   the witness, FFT each advice column into coefficient form,
   blind, commit, write the commitment to the transcript.
3. Squeeze the lookup challenge $\theta$; run the lookup prover
   for each lookup argument, commit $a'$ and $s'$.
4. Squeeze $\beta, \gamma$; run the permutation prover and the
   second half of the lookup prover (the grand product $z$).
5. Commit to vanishing polynomial commitments.
6. Squeeze the gate-combining challenge $y$.
7. Construct the quotient polynomial $h$ on the extended coset
   domain; divide by $t_H$; split into pieces of length $n$;
   commit to each piece.
8. Squeeze the opening challenge $x$.
9. Evaluate every polynomial at $x$ (and at $\omega \cdot x$ for
   rotated queries); write evaluations to the transcript.
10. Run the multiopen prover to produce a single IPA opening.

### 3.3 The verifier

```rust file=../../halo2_proofs/src/plonk/verifier.rs#L65-L120 title="halo2_proofs/src/plonk/verifier.rs"
```
The verifier is structurally a mirror of the prover:

1. Hash the verifying key.
2. Read the advice / lookup / permutation commitments,
   re-squeeze each Fiat-Shamir challenge from the transcript.
3. Read the vanishing-polynomial commitments and the quotient
   polynomial commitments.
4. Read all polynomial evaluations.
5. Re-compute the expected value of $h(x) \cdot t_H(x)$ from the
   gate-combining identity and check it against the supplied
   evaluations.
6. Run the multiopen verifier, producing a `Guard` whose
   final-MSM the strategy then resolves.

The `VerificationStrategy` parameter (see
[`halo2_proofs/src/plonk/verifier.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/verifier.rs))
selects between single (default) and batched verification.
`SingleVerifier` forces the MSM immediately; `BatchVerifier`
accumulates the MSM across many proofs and forces them all once
at the end. Batch verification can drop per-proof verifier cost
from $O(n)$ to $O(\log n)$ amortized.

### 3.4 The integration test

```rust file=../../halo2_proofs/tests/plonk_api.rs#L1-L60 title="halo2_proofs/tests/plonk_api.rs"
```
The integration test in `tests/plonk_api.rs` runs the full
keygen-prove-verify loop on a hand-designed circuit and compares
the resulting proof bytes against the checked-in fixture
`plonk_api_proof.bin`. If you change anything that affects proof
bytes (a new commitment, a reordered challenge, even an extra
zero pad), this fixture must be regenerated.

## 4. Failure modes

- **Forgetting to hash the VK first.** The first thing both
  prover and verifier do is `vk.hash_into(transcript)`. Skipping
  this would let a malicious prover replay a proof against a
  different circuit. Audit any new transcript path for this.
- **Mismatched `Params`.** Prover and verifier must use the same
  `Params(k)`. Halo 2's
  [`Params::new`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment.rs)
  is deterministic, so a fresh prover and verifier rederive
  identical params; do not introduce randomness here.
- **Proof byte drift.** Changing the order of any operation that
  writes to the transcript (commitments, evaluations) silently
  changes the proof bytes. The `plonk_api_proof.bin` fixture
  catches this in CI.
- **Skipping the `Guard`.** If you write a custom verifier that
  drops the `Guard` returned by `verify_proof` without calling
  `use_challenges`, the multiopen IPA check never runs and every
  proof passes. This is the kind of bug a security review catches.

## 5. Spec pointers

- [Halo paper, eprint 2019/1021](https://eprint.iacr.org/2019/1021)
  section 4: the full IOP-to-NIZK construction.
- [PLONK paper, eprint 2019/953](https://eprint.iacr.org/2019/953)
  section 5: the integration of the gate, permutation, and lookup
  arguments. The halo2 prover follows the same template.
- The
  [Halo 2 Book Protocol Description chapter](https://zcash.github.io/halo2/design/protocol.html)
  walks through the same flow at the spec level.

## 6. Exercises

1. Open
   [`halo2_proofs/src/plonk/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/prover.rs)
   and find every call to `transcript.common_point(...)`,
   `transcript.write_point(...)`, and `transcript.write_scalar(...)`.
   In order, list each. This is the proof's structure.
2. Repeat the exercise for the verifier in
   [`halo2_proofs/src/plonk/verifier.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/verifier.rs)
   using `read_point` and `read_scalar`. Confirm the two lists
   are dual.
3. Run `cargo test --release -p halo2_proofs --test plonk_api` and
   observe the test name(s). Confirm the proof fixture matches.
4. Modify one byte of
   [`halo2_proofs/tests/plonk_api_proof.bin`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/tests/plonk_api_proof.bin)
   (use a hex editor) and rerun the test. Confirm the verifier
   rejects.
5. Why does `keygen_vk` need a witness-free circuit, while
   `keygen_pk` accepts a witness-bearing one? Answer in two
   sentences after reading both functions.

### Answers in the code

- Exercise 1: the order is (roughly) `vk.hash_into`, advice
  commitments, lookup permuted commitments, permutation product
  commitments, vanishing commitments, quotient-poly piece
  commitments, evaluations, multiopen openings.
- Exercise 5: `keygen_vk` only needs the shape; the witness would
  be redundant. `keygen_pk` re-synthesizes to fill the extended
  caches but does not include witness data in the proving key.

## 7. Further reading

- The
  [Halo 2 Book Implementation chapter](https://zcash.github.io/halo2/design/implementation.html)
  for the engineering details that drove the prover layout.
- The
  [Halo 2 Book Proofs chapter](https://zcash.github.io/halo2/design/implementation/proofs.html)
  for an opcode-by-opcode description of the proof format.
