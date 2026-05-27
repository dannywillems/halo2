---
sidebar_position: 3
title: Field Arithmetic, FFT, and MSM
description: The pasta field, the radix-2 FFT, Pippenger's multi-scalar multiplication, and the parallelization helper.
---

# Field Arithmetic, FFT, and MSM

## 1. Why this chapter exists

Everything halo2 does, from polynomial commitment to verifier
challenges, eventually bottoms out in one of three primitives:

- field arithmetic on $\mathbb{F}_p$ (the Pallas / Vesta scalar
  field),
- the radix-2 number-theoretic transform (NTT) that we will simply
  call FFT,
- multi-scalar multiplication on the corresponding curve,
  $\sum_i k_i G_i$.

If you change any of them and the constants do not match the
formal spec, every test in the suite passes and the verifier still
accepts forged proofs. That is the failure mode the maintainers
worry about, and it is the failure mode CI cannot catch on its own.
This chapter tells you which file is the law for each primitive.

## 2. Definitions

**Definition (Pasta fields).** Pallas and Vesta are a 2-cycle of
prime-order curves defined in
[`pasta_curves`](https://github.com/zcash/pasta_curves). Let $p$ and
$q$ be the two primes, with $p \approx 2^{255}$ and $q \approx 2^{255}$.
The Pallas base field $\mathbb{F}_p$ is the Vesta scalar field, and
vice versa. This 2-cycle is what makes recursive proof composition
practical. Both primes have $|\mathbb{F}^{\times}|$ divisible by
$2^{32}$, so $\mathbb{F}$ contains primitive $2^{32}$-th roots of
unity; the FFT supports any $n = 2^k$ for $k \leq 32$.

**Definition (Multiplicative subgroup of order $n$).** Let
$n = 2^k$ and let $\omega \in \mathbb{F}_p^\times$ be a primitive
$n$-th root of unity. The set
$H = \{1, \omega, \omega^2, \dots, \omega^{n-1}\}$ is the
multiplicative subgroup of order $n$. The PLONKish circuit lives
on $H$: row $i$ corresponds to the point $\omega^i$.

**Definition (FFT).** Given coefficients $(a_0, \dots, a_{n-1})$
of a polynomial $a(X) = \sum_i a_i X^i$, the FFT computes the
evaluations $(a(\omega^0), \dots, a(\omega^{n-1}))$ in
$O(n \log n)$ field operations. The radix-2 Cooley-Tukey FFT used
here recurses on $n / 2$ and combines with the butterfly
$\bigl(a(\omega^i), a(\omega^{i+n/2})\bigr) = \bigl(e(\omega^{2i}) + \omega^i o(\omega^{2i}), e(\omega^{2i}) - \omega^i o(\omega^{2i})\bigr)$
where $e$ and $o$ are the even-index and odd-index polynomials.

**Definition (MSM, Pippenger's algorithm).** Given scalars
$k_1, \dots, k_m \in \mathbb{F}_p$ and points
$G_1, \dots, G_m \in \mathbb{G}$, the multi-scalar multiplication
problem is to compute $\sum_i [k_i] G_i$. Pippenger's algorithm
windows the scalars into $w$-bit chunks, accumulates each chunk's
contribution into $2^w$ "buckets", reduces the buckets, and
combines across windows. The cost is roughly $m / \log m$ point
operations plus $O(\lambda)$ for finalization, where $\lambda$ is
the bit length of the scalars.

## 3. The code

### 3.1 The module surface

```rust file=../../halo2_proofs/src/arithmetic.rs#L1-L27 title="halo2_proofs/src/arithmetic.rs"
```
`pub use pasta_curves::arithmetic::*;` re-exports the curve traits
(`CurveAffine`, `CurveExt`, `Group`, `Coordinates`, etc.) so the
rest of the crate can refer to them without naming
`pasta_curves` directly.

### 3.2 Multi-scalar multiplication

The fast path uses Pippenger:

```rust file=../../halo2_proofs/src/arithmetic.rs#L141-L180 title="halo2_proofs/src/arithmetic.rs"
```
A few invariants worth knowing:

- The window size $w$ is chosen as a function of $m$. The code
  uses a heuristic of $w \approx \log_2 m - 3$ for $m \geq 32$,
  falling back to $w = 3$ for tiny inputs.
- `coeffs` and `bases` must have equal length; the function does
  not check this on the hot path. Length mismatch is a programmer
  bug.
- For very small inputs the dispatch falls through to
  `small_multiexp`, a naive double-and-add:

```rust file=../../halo2_proofs/src/arithmetic.rs#L114-L136 title="halo2_proofs/src/arithmetic.rs"
```
The 0.3.2 changelog notes "performance has been improved" for
`best_multiexp`; PR
[#796](https://github.com/zcash/halo2/pull/796) is the source of
that improvement.

### 3.3 The FFT

```rust file=../../halo2_proofs/src/arithmetic.rs#L188-L255 title="halo2_proofs/src/arithmetic.rs"
```
The dispatcher `best_fft` chooses between a serial recursive
implementation and a parallel one based on `log_n` and the number
of rayon threads. The recursion is in `recursive_butterfly_arithmetic`:

```rust file=../../halo2_proofs/src/arithmetic.rs#L256-L295 title="halo2_proofs/src/arithmetic.rs"
```
The slice `a` is mutated in place; on entry it holds bit-reversed
coefficients, on exit it holds evaluations in standard order.

### 3.4 Parallelization helper

```rust file=../../halo2_proofs/src/arithmetic.rs#L341-L363 title="halo2_proofs/src/arithmetic.rs"
```
The closure receives a mutable subslice and the global index of its
first element. This is the workhorse for almost every parallel
loop in the crate. Always prefer it to `chunks_mut` plus
`rayon::join` because it respects the `multicore` feature flag.

### 3.5 Other small but load-bearing helpers

- `eval_polynomial`: Horner's method for evaluating
  $\sum_i a_i x^i$.
- `compute_inner_product`: $\sum_i a_i b_i$, used in the IPA
  prover.
- `kate_division`: divide a polynomial by $X - b$, used inside the
  multiopen prover.
- `lagrange_interpolate`: given $(x_i, y_i)$ pairs, recover the
  polynomial. Used in tests and the IPA setup.

## 4. Failure modes

- **Wrong root of unity.** If a caller passes the wrong $\omega$
  to `best_fft`, the polynomial in/out conversion silently
  corrupts and every downstream commitment is wrong. The
  `EvaluationDomain` type (chapter 04) exists exactly to avoid
  this; never call `best_fft` directly outside `poly/domain.rs`
  unless you absolutely know what you are doing.
- **Edge case in `best_multiexp` for $m = 0$.** The function
  returns the curve identity. The change in
  [#796](https://github.com/zcash/halo2/pull/796) required adding
  the test
  [`test_create_proof`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs)
  precisely to lock that behaviour in.
- **Forgetting `--features multicore`.** Without it, every parallel
  helper degrades to serial. The CI default has multicore enabled;
  benchmarks without it can be 8 to 16x slower.
- **Bit-reversal mismatch.** The recursive FFT expects bit-reversed
  input; if you ever call it from a new code path, you must
  bit-reverse the input first (see how
  [`EvaluationDomain`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs)
  does it).

## 5. Spec pointers

- The Pasta cycle and the choice of primes are documented in the
  [`pasta_curves` README](https://github.com/zcash/pasta_curves)
  and in
  [Daira Hopwood's explainer](https://github.com/daira/pasta).
- The Cooley-Tukey FFT is described in any algorithms textbook;
  for the field-theoretic NTT view, see chapter 30 of
  [CLRS](https://mitpress.mit.edu/9780262046305/introduction-to-algorithms/).
- Pippenger's algorithm: Pippenger, "On the evaluation of powers
  and monomials", SIAM J. Comput. (1976). For a modern exposition
  in the SNARK setting, see Bernstein et al.,
  [eprint 2020/1145](https://eprint.iacr.org/2020/1145).

## 6. Exercises

1. Open
   [`halo2_proofs/src/arithmetic.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs)
   and identify the function that chooses the Pippenger window
   size. Write down the formula it uses as a function of `m`.
2. Find the test at the bottom of
   [`halo2_proofs/src/arithmetic.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs)
   that checks `best_multiexp` against `small_multiexp` and run it:

   ```bash
   cargo test --release -p halo2_proofs arithmetic
   ```

3. Add an extra `#[test]` to
   [`halo2_proofs/src/arithmetic.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs)
   verifying that `best_multiexp` on an empty input returns the
   curve identity. Run the test and confirm it passes.
4. Run the FFT benchmark:

   ```bash
   cargo bench -p halo2_proofs --bench fft
   ```

   Observe how the per-operation cost scales with `log_n`; the
   slope should be roughly linear (cost is $O(\log n)$ per element).

### Answers in the code

- Exercise 1: the window size selection is at the top of
  `best_multiexp` in
  [`halo2_proofs/src/arithmetic.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs).
- Exercise 2: the matching test is named `test_best_multiexp` (or
  similar) inside the `#[cfg(test)] mod` at the bottom of the same
  file.

## 7. Further reading

- The
  [`pasta_curves`](https://github.com/zcash/pasta_curves) source
  for the curve implementations themselves.
- The
  [`group`](https://docs.rs/group) and
  [`ff`](https://docs.rs/ff) traits, which are the abstract
  interfaces halo2 codes against.
