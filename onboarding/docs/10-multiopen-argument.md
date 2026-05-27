---
sidebar_position: 10
title: The Multiopen Argument
description: Compressing many polynomial-opening claims into one IPA opening, and the 2025 soundness fix.
---

# The Multiopen Argument

## 1. Why this chapter exists

A halo2 verifier wants to check evaluations of dozens of
polynomials at a handful of points. Running an IPA opening per
polynomial would be both slow and large. The multiopen argument
groups the polynomials by their query points, linearly combines
them with verifier challenges, and runs a single IPA on the
collapsed result. The argument is small in code, large in
consequences: the only critical CVE-class fix in halo2's history
landed here.

## 2. Definitions

**Definition (Polynomial query).** A triple $(p, x, e)$ where $p$
is a committed polynomial, $x$ is the point at which it is
opened, and $e = p(x)$ is the claimed evaluation.

**Definition (Multiopen instance).** A finite list of polynomial
queries. The prover and verifier both walk this list using the
same canonical order; any reordering must be agreed on by both
sides (this is what the soundness fix below is about).

**Definition (Point set, commitment-keyed grouping).** Group the
queries first by the set of points $X = \{x_1, \dots, x_m\}$ at
which a given polynomial is opened, and then by the polynomial
commitment. The "rotation set" for a polynomial is the multiset of
points at which it is opened.

**Definition (Quotient polynomial $q$).** For each polynomial $p$
opened at points $X$, define the quotient
$q(X) = (p(X) - r(X)) / Z_X(X)$ where $r$ is the polynomial that
interpolates the $(x_i, e_i)$ pairs and $Z_X(X) = \prod_{x_i \in X} (X - x_i)$.
The multiopen argument reduces to the IPA opening of a single
random linear combination of all such $q$'s.

## 3. The code

### 3.1 Module structure and challenges

```rust file=../../halo2_proofs/src/poly/multiopen.rs#L1-L40 title="halo2_proofs/src/poly/multiopen.rs"
```
Four named challenges, in order of use:

- $x_1$: collapses queries that share the same point set.
- $x_2$: keeps the per-point-set quotient polynomials linearly
  independent.
- $x_3$: the eventual opening point.
- $x_4$: collapses the remaining openings at $x_3$ before the
  final IPA.

The order matters: each is squeezed from the transcript only
after all the commitments it depends on have been written. Re-read
this when chasing a multiopen verifier mismatch.

### 3.2 Query types

```rust file=../../halo2_proofs/src/poly/multiopen.rs#L41-L120 title="halo2_proofs/src/poly/multiopen.rs"
```
`ProverQuery` carries the actual polynomial coefficients; the
prover needs them. `VerifierQuery` carries only the commitment
reference and the claimed evaluation; the verifier only ever sees
those.

### 3.3 The prover

```rust file=../../halo2_proofs/src/poly/multiopen/prover.rs#L20-L120 title="halo2_proofs/src/poly/multiopen/prover.rs"
```
The prover's flow:

1. Bucket the queries by point set; within each bucket, compute
   a linear combination of polynomials using powers of $x_1$.
2. Compute the per-bucket quotient
   $q = (p - r) / Z_X$ using `kate_division`.
3. Combine all $q$'s using powers of $x_2$ into a single $f$.
4. Commit to $f$.
5. Squeeze $x_3$; evaluate $f$ at $x_3$ and write the evaluations
   of every $p$ at every $x_i$.
6. Squeeze $x_4$; collapse all the openings at $x_3$ into one
   final IPA opening.

### 3.4 The verifier

```rust file=../../halo2_proofs/src/poly/multiopen/verifier.rs#L1-L80 title="halo2_proofs/src/poly/multiopen/verifier.rs"
```
The verifier mirrors the prover step by step, replacing
"polynomial" with "commitment + evaluation" everywhere, and runs
the final IPA verifier on the collapsed claim.

### 3.5 The 2025 soundness fix (halo2_proofs 0.3.1)

```markdown file=../../halo2_proofs/CHANGELOG.md#L30-L45 title="halo2_proofs/CHANGELOG.md"
```
The pre-0.3.1 multiopen code grouped queries by
`(point, commitment)` using a hash map that did not detect when
the *same* `(point, commitment)` pair was inserted twice with two
different evaluations. A buggy circuit could submit
`(C, x, e_1)` and `(C, x, e_2)` with $e_1 \neq e_2$; the
verifier accepted both. The fix has three parts:

1. The multiopen code now uses `indexmap::IndexMap` to preserve
   insertion order deterministically.
2. Both prover and verifier explicitly check that no
   `(point, commitment)` pair is inserted twice with conflicting
   evaluations. The mismatch returns `Error::OpeningError`.
3. The corresponding test in
   [`halo2_proofs/tests/plonk_api.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/tests/plonk_api.rs)
   was extended to lock the new behaviour.

The bug was reported by Suneal at zkSecurity. Note: the impacted
proofs were only insecure when the prover was already buggy; a
correct circuit was never affected. The fix is still a hard
requirement because it preserves soundness against adversarial
provers.

## 4. Failure modes

- **Inserting `(commitment, point, evaluation)` triples in
  non-deterministic order.** Pre-0.3.1, this changed proof bytes
  silently. Post-0.3.1, it returns `Error::OpeningError` on the
  prover side; do not catch this error.
- **Sharing a query type between prover and verifier without
  rerunning the bucketing.** The verifier rebuilds the buckets
  from scratch using the same `(point, commitment)` keys. If your
  prover deduplicated the queries but your verifier did not, you
  will get a verifier mismatch.
- **Forgetting `kate_division` on a non-monic divisor.**
  `kate_division` assumes division by $X - b$. The multiopen
  prover uses it on the polynomial $Z_X(X)$ that is *not*
  $X - b$; the prover instead uses an explicit division by
  evaluating $Z_X$ on the extended domain and inverting
  pointwise. Read the prover code carefully before extending it.

## 5. Spec pointers

- [Halo paper, eprint 2019/1021](https://eprint.iacr.org/2019/1021),
  section "Polynomial commitments" includes the original
  multiopen construction.
- The Halo 2 Book
  [Multipoint opening argument chapter](https://zcash.github.io/halo2/design/proving-system/multipoint-opening.html)
  derives the same construction with worked notation.
- The zkSecurity write-up of the 2025 disclosure (search
  "zkSecurity halo2 multiopen"); the GitHub advisory is
  [GHSA-fmqx-q3p4-mrj9](https://github.com/zcash/halo2/security/advisories/GHSA-fmqx-q3p4-mrj9).

## 6. Exercises

1. Open
   [`halo2_proofs/src/poly/multiopen.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/multiopen.rs)
   and find each of the four challenge type aliases
   $x_1, x_2, x_3, x_4$. Map each to the step in section 3.3
   above.
2. Read
   [`halo2_proofs/CHANGELOG.md`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/CHANGELOG.md)'s
   `0.3.1` entry. Identify the version's full security text and
   note the role `indexmap` plays in the fix.
3. Write a small Rust harness that asks `create_proof` to open
   the same `(point, commitment)` twice with different
   evaluations. Confirm that the current code returns
   `Error::OpeningError`. (Hint: the easiest way is to construct
   two `ProverQuery` values directly and call
   `create_proof` from `poly::multiopen` with them.)
4. The prover commits to a single polynomial $f$ that combines
   every bucket. Why does this give a proof size of
   $O(\log n)$ in $n$ rather than $O((\text{number of buckets}) \log n)$?

### Answers in the code

- Exercise 1: $x_1$ at line ~22, $x_2$ at line ~27, $x_3$ at line
  ~32, $x_4$ at line ~37 in
  [`halo2_proofs/src/poly/multiopen.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/multiopen.rs).
- Exercise 4: combining the per-bucket quotient polynomials into
  a single $f$ collapses everything to one polynomial of degree
  $\leq n$; only one IPA opening is required.

## 7. Further reading

- The
  [Halo 2 Book chapter on the multipoint opening argument](https://zcash.github.io/halo2/design/proving-system/multipoint-opening.html).
- The
  [halo2 GitHub Security Advisory GHSA-fmqx-q3p4-mrj9](https://github.com/zcash/halo2/security/advisories/GHSA-fmqx-q3p4-mrj9)
  for the full disclosure timeline.
