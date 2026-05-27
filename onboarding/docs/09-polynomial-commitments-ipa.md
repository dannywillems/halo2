---
sidebar_position: 9
title: Polynomial Commitments and the Inner Product Argument
description: Pedersen commitments to polynomials and the logarithmic-size inner product argument used to open them.
---

# Polynomial Commitments and the IPA

## 1. Why this chapter exists

A PLONKish proof is, structurally, a list of polynomials together
with their evaluations at a few challenge points. The polynomial
commitment scheme is what makes the proof short: instead of sending
the polynomials, the prover sends $O(\log n)$ group elements plus
the evaluations, and the verifier checks the evaluations against
the commitments. halo2 uses the inner product argument (IPA) from
the Halo paper, which is trustless and recursion-friendly. Two
practical reasons this chapter matters: (a) every other proof
component eventually feeds into the IPA, so knowing its interface
is non-negotiable; (b) the IPA's verifier cost is the dominant
verification cost for small circuits.

## 2. Definitions

Fix the small domain $H$ with $n = 2^k$ and a cyclic group
$\mathbb{G}$ of prime order $q$.

**Definition (Pedersen vector commitment).** Given generators
$G_0, \dots, G_{n-1}, W \in \mathbb{G}$ chosen by hashing to the
curve (so no one knows discrete logs), a commitment to a vector
$\vec a \in \mathbb{F}_q^n$ with blinding $r \in \mathbb{F}_q$ is
$$\mathsf{Com}(\vec a; r) = \sum_{i=0}^{n-1} a_i G_i + r W \in \mathbb{G}.$$
This is computationally binding and perfectly hiding.

**Definition (Pedersen polynomial commitment).** For a polynomial
$p(X) = \sum a_i X^i$, define
$\mathsf{Com}(p; r) = \mathsf{Com}(\vec a; r)$. Halo 2's `Params`
also carry a Lagrange-basis generator set
$L_0, \dots, L_{n-1}$ so the same commitment can be expressed in
either basis without an extra FFT.

**Definition (Inner product argument).** A succinct argument that
$\langle \vec a, \vec b \rangle = c$ for a publicly known $\vec b$
and a committed $\vec a$. The argument is $O(\log n)$ rounds and
sends $2 \log n + 2$ group elements; verification time is also
$O(\log n)$ after a single multi-scalar multiplication that can
be deferred or amortized across many proofs.

**Definition (IPA polynomial opening).** Opening a polynomial
commitment $\mathsf{Com}(p)$ at a point $x$ reduces to an inner
product check: $p(x) = \langle \vec a, \vec b \rangle$ with
$b_i = x^i$. The IPA then certifies the inner product.

**Definition (Accumulator).** The verifier of the IPA does not
finalize the multi-scalar check immediately; it can produce an
*accumulator* (commitment $G' \in \mathbb{G}$ plus claimed scalar
$c' \in \mathbb{F}_q$) such that the entire verification reduces
to checking $G' = \sum_i c'_i G_i$. Halo's recursion uses this:
the inner circuit "verifies" the previous proof by absorbing the
accumulator and the outer verifier checks the combined
accumulator at the end. This is the trick that lets halo2 do
trustless recursion.

## 3. The code

### 3.1 The setup struct `Params`

**Source:** [`halo2_proofs/src/poly/commitment.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment.rs#L23-L45)

`Params::new(k)` deterministically derives the generators by
hashing-to-curve with the personalization `"Halo2-Parameters"`.
Two consequences:

- There is no trusted setup; anyone can rederive `Params(k)` and
  confirm they match by recomputing the hash.
- The setup is per-`k`: a circuit of degree $2^{k}$ uses a
  `Params(k')` for some $k' \geq k$.

The struct holds:

- `g`: the $n$ generators in coefficient-basis form.
- `g_lagrange`: the same $n$ generators rotated by the inverse FFT
  matrix, so that committing a Lagrange-basis polynomial is also
  one MSM.
- `w`: the blinding generator $W$.
- `u`: the generator used to absorb the inner product (introduced
  by Halo's IPA variant; this is the "challenge group element"
  that lets the prover commit to $c$ alongside $\vec a$).

### 3.2 The commitment operation

`Params::commit` is the workhorse:

**Source:** [`halo2_proofs/src/poly/commitment.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment.rs#L115-L165)

It is just `best_multiexp(values, g)` plus a blinding term
`r * w`. The Lagrange variant uses `g_lagrange` instead of `g`.

### 3.3 The `Blind` newtype

**Source:** [`halo2_proofs/src/poly/commitment.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment.rs#L208-L255)

`Blind<F>` is a thin wrapper that overloads `+`, `*`, etc. so that
the prover can do "blinding arithmetic" with the same expressions
it uses for the underlying polynomials. The Pedersen blinding for
$\alpha p_1 + \beta p_2$ is $\alpha r_1 + \beta r_2$; the
`Blind` newtype enforces that algebra at the type level.

### 3.4 The IPA prover

**Source:** [`halo2_proofs/src/poly/commitment/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment/prover.rs)

The protocol, in pseudocode:

```
inputs: a (length n), b (length n) with b_i = x^i, blinding r, commitment P = Com(a; r)
for round = 0..log n:
    split a, b, G into left/right halves (a_l, a_r), (b_l, b_r), (G_l, G_r)
    L = <a_l, G_r> + <a_l, b_r> * U + blinding_l * W
    R = <a_r, G_l> + <a_r, b_l> * U + blinding_r * W
    write L, R to transcript
    squeeze challenge u from transcript
    a' = a_l + u^{-1} a_r; b' = b_l + u * b_r; G' = G_l + u * G_r
    a, b, G = a', b', G'
output: final scalar a (of length 1) and final blinding
```

The verifier reconstructs $G'$ from $(G_0, \dots, G_{n-1})$ and
the round challenges $u_0, \dots, u_{k-1}$. The Halo paper's trick
is that this reconstruction is itself an MSM with structured
scalars, so the verifier can either compute it directly or defer
it into an accumulator.

### 3.5 The IPA verifier

**Source:** [`halo2_proofs/src/poly/commitment/verifier.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment/verifier.rs)

The verifier returns a `Guard` rather than a boolean. The `Guard`
holds the pending checks; calling `Guard::use_challenges` runs the
final MSM. The `BatchVerifier` aggregates many `Guard`s into one
MSM, dropping per-proof cost from $O(n)$ to $O(\log n)$ amortized.

### 3.6 The MSM accumulator

**Source:** [`halo2_proofs/src/poly/commitment/msm.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment/msm.rs)

`MSM` is a "deferred MSM" type: the verifier records
$(\text{base}, \text{scalar})$ pairs and runs the actual MSM only
when forced. This is what enables `BatchVerifier`.

## 4. Failure modes

- **Reusing `Params` across `k` values.** `Params(k)` is only valid
  for circuits up to size $2^k$. Reusing a smaller `Params` for a
  larger circuit truncates the witness silently; reusing a larger
  `Params` works but wastes setup.
- **Forgetting to blind.** Calling `params.commit(values,
  Blind::default())` (i.e. with $r = 0$) makes the commitment
  binding but not hiding. The IPA proof itself becomes
  non-zero-knowledge. Always pass a freshly random `Blind` for
  any witness polynomial.
- **Mismatched basis.** `commit` and `commit_lagrange` use
  different generator sets. Passing a Lagrange-basis polynomial
  to `commit` (or vice versa) gives you a commitment to a
  different polynomial. The `Polynomial<F, B>` type tag prevents
  this at the boundary.
- **Forgetting `Guard::use_challenges`.** A `Guard` that is
  dropped without being finalized silently passes verification.
  Either call `use_challenges` or combine it into a
  `BatchVerifier`.

## 5. Spec pointers

- [Halo paper, eprint 2019/1021](https://eprint.iacr.org/2019/1021):
  the original construction of the IPA with the accumulator trick.
- [Bowe and Hopwood, "Faster batch forgery identification"](https://eprint.iacr.org/2018/280): batch verification of IPA.
- [Bunz et al., "Bulletproofs"](https://eprint.iacr.org/2017/1066):
  the inner-product argument's lineage and the recursive halving
  trick.

## 6. Exercises

1. Open
   [`halo2_proofs/src/poly/commitment.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment.rs)
   and find the personalization string used by `Params::new`. What
   would change if you altered it?
2. The IPA proof contains $2 \log_2 n + 2$ field / group elements.
   Count them in the prover code: trace one `for round` loop in
   [`halo2_proofs/src/poly/commitment/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment/prover.rs)
   and identify which lines write `L` and `R` to the transcript.
3. Run the `msm` bench:

   ```bash
   cargo bench -p halo2_proofs --bench msm
   ```

   How does single-proof verification cost compare with the
   `plonk` bench's amortized batched cost?
4. Implement a `print_size` helper that, given a proof byte slice,
   prints `n_round_messages = (len - constant_part) / point_size`.
   Use it on a proof produced by the `simple-example`.

### Answers in the code

- Exercise 1: the personalization is the byte string
  `"Halo2-Parameters"` passed to `C::CurveExt::hash_to_curve`.
  Changing it would shift every generator; all existing
  verifying keys would be invalidated.
- Exercise 2: the `L` and `R` writes are in the inner loop of
  `create_proof` in
  [`halo2_proofs/src/poly/commitment/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/commitment/prover.rs).

## 7. Further reading

- The
  [Halo 2 Book "Inner product argument" chapter](https://zcash.github.io/halo2/design/proving-system/inner-product.html)
  goes through the math in linear order, matching the variable
  names used in the code.
- The Plonk2 / halo2 talk at
  [Zcon3](https://www.youtube.com/watch?v=4fGGmqDfNyc) covers the
  recursion-friendly aspect of this commitment scheme.
