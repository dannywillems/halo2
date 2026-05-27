---
sidebar_position: 4
title: Polynomial Domains and Basis Conversions
description: EvaluationDomain, the coefficient / Lagrange / extended-Lagrange bases, and the rotation convention.
---

# Polynomial Domains and Basis Conversions

## 1. Why this chapter exists

Inside halo2 a "polynomial" is not just a vector of field elements;
it is a vector tagged with a *basis*. The same `Vec<F>` can mean
"coefficients of $\sum_i a_i X^i$", "evaluations on a small domain
$H$ of size $n$", or "evaluations on an extended coset domain". If
you ever confuse the bases, the FFT will silently corrupt your data
and the resulting commitment will be wrong, with no error. This
chapter is the contract that `EvaluationDomain` enforces.

## 2. Definitions

Fix $n = 2^k$ and let $\omega \in \mathbb{F}_p^\times$ be a
primitive $n$-th root of unity. Let
$H = \{1, \omega, \dots, \omega^{n-1}\}$.

**Definition (Coefficient basis).** The polynomial $p$ is given by
the coefficients $(a_0, \dots, a_{n-1})$ of
$p(X) = \sum_{i=0}^{n-1} a_i X^i$. Tag: `Coeff`.

**Definition (Lagrange basis on $H$).** The polynomial $p$ is given
by the evaluations $(p(\omega^0), \dots, p(\omega^{n-1}))$. Tag:
`LagrangeCoeff`. The corresponding basis polynomials are
$L_i(X)$ with $L_i(\omega^j) = \delta_{ij}$.

**Definition (Extended Lagrange basis).** Let $\zeta \in
\mathbb{F}_p$ be a non-trivial cube root of unity. Let
$n' = 2^{k'}$ with $k' = k + \log_2 d$, where $d$ is the
quotient-polynomial degree. The extended coset domain is
$\zeta \cdot \langle \omega' \rangle$ for a primitive $n'$-th root
of unity $\omega'$. The Lagrange representation of a polynomial on
this coset is its tag `ExtendedLagrangeCoeff`. The quotient
polynomial $t(X)$ is the only object that natively lives here.

**Definition (Rotation).** A `Rotation(r)` is an integer offset.
Inside a gate, querying advice column $A$ at `Rotation::next()`
(that is, $r = 1$) means evaluating $A(\omega \cdot X)$. In Lagrange
basis this just shifts the index by $r$ modulo $n$.

**Definition (Vanishing polynomial of $H$).** The polynomial
$t_H(X) = X^n - 1$. It is zero on every element of $H$. The
prover constructs a quotient $h(X) = t(X) / t_H(X)$ where $t(X)$
is the row-vanishing polynomial of all gate constraints; the
verifier checks that the resulting identity holds at a random
challenge point.

## 3. The code

### 3.1 The struct and the constants it caches

**Source:** [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L19-L36)

Every value here is a precomputed function of $k$ and the
quotient-degree bound $j$:

- `n = 2^k`, the small-domain size.
- `omega`, `omega_inv`: the primitive $n$-th root and its inverse.
- `extended_k`, `extended_omega`, `extended_omega_inv`: the
  extended domain.
- `g_coset = zeta`, `g_coset_inv = zeta^2`: the coset shift.
- `t_evaluations`: $t_H$ evaluated on the extended coset, ready to
  be divided by.
- `ifft_divisor = 1/n` and `extended_ifft_divisor = 1/n'`: the
  inverse-FFT scaling factors.
- `barycentric_weight`: $1 / \prod_{i \neq 0} (1 - \omega^i)$, used
  in the multiopen prover.

### 3.2 Construction

**Source:** [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L37-L148)

Two invariants this code enforces:

- $2^{\text{extended\_k}} \geq n \cdot j$, so the quotient
  polynomial fits in the extended domain.
- `extended_k <= F::S`, where `F::S` is the largest $s$ such that
  $\mathbb{F}^\times$ has an element of order $2^s$. For Pallas /
  Vesta, `S = 32`. Going past this is a constructor panic, never a
  silent failure.

### 3.3 Basis conversions

The public conversions form a small commutative-ish diagram:

```
LagrangeCoeff  --lagrange_to_coeff-->  Coeff
                                         |
                                         | coeff_to_extended
                                         v
                              ExtendedLagrangeCoeff
                                         |
                                         | extended_to_coeff
                                         v
                                       Vec<F>  (coefficient form,
                                                returned as raw Vec)
```

The methods:

- [`empty_lagrange`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L181)
  and `empty_coeff`: allocate a zeroed `Polynomial<F, _>` of the
  right length and tag.
- [`lagrange_to_coeff`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L227):
  inverse FFT on the small domain.
- [`coeff_to_extended`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L241):
  pad to length $n'$, multiply by powers of $\zeta$ to move to the
  coset, FFT on the extended domain.
- [`extended_to_coeff`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L303):
  inverse FFT on the extended domain, divide out the coset shift.

The fact that the return type of `extended_to_coeff` is `Vec<F>`
rather than a tagged `Polynomial<F, Coeff>` is deliberate: the
result is meant to be sliced into pieces of length $n$ for the
chunked-quotient construction.

### 3.4 The vanishing-polynomial division

**Source:** [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L327-L350)

This is where the cached `t_evaluations` are used: dividing a
polynomial in `ExtendedLagrangeCoeff` by $t_H$ is a pointwise
multiplication by `1 / t_H(zeta * omega'^i)`.

### 3.5 Rotation

**Source:** [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs#L406-L419)

`rotate_omega(value, rotation)` returns $\text{value} \cdot
\omega^{\text{rotation}}$. In the verifier this is how the random
challenge $x$ becomes $\omega^r \cdot x$ for a query at relative
offset $r$.

## 4. Failure modes

- **Cross-basis arithmetic.** Adding a `Polynomial<F, Coeff>` and a
  `Polynomial<F, LagrangeCoeff>` is a type error; the tags are
  there to prevent this. Do not work around the tags by using raw
  vectors.
- **Wrong $j$.** `EvaluationDomain::new(j, k)` panics if `j` is too
  small to host the quotient. A common bug when extending the
  arithmetization with a new degree-$d'$ gate: forgetting to
  recompute the max-degree bound passed as `j`.
- **`extended_to_coeff` returning a raw `Vec`.** The caller is
  expected to slice it into chunks; treating the whole vector as a
  single polynomial of length $n'$ will produce a polynomial of
  the wrong degree.
- **Domain reuse across circuits.** Two different circuits with
  the same `(j, k)` *can* share an `EvaluationDomain`. Halo 2 does
  not, on the assumption that key generation rebuilds the domain
  every time. Do not optimize this away without checking the call
  sites in
  [`plonk/keygen.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/keygen.rs).

## 5. Spec pointers

- The
  [Halo 2 Book "Polynomials" background chapter](https://zcash.github.io/halo2/background/polynomials.html)
  walks through the same coefficient / Lagrange / coset basis story
  with worked examples.
- PLONK's notion of the extended evaluation domain is described in
  section 4 of
  [eprint 2019/953](https://eprint.iacr.org/2019/953).
- Bowe and Hopwood's note "Faster batch forgery identification"
  (and the related blog posts) explains the choice of $\zeta$ as
  the coset shift.

## 6. Exercises

1. Open
   [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs)
   and identify the assertion `extended_k <= F::S`. What value
   does `F::S` take for the Pallas scalar field?
2. Trace what happens to a single value as it goes through
   `coeff_to_extended` followed by `extended_to_coeff`. Specifically,
   write down the four operations applied in order and confirm
   that they compose to the identity on the first $n$ entries.
3. Add a `#[test]` to
   [`halo2_proofs/src/poly/domain.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/domain.rs)
   building an `EvaluationDomain<Fp>::new(2, 4)` and checking that
   `get_omega().pow_vartime([16])` equals one. Run it.
4. The function `rotate_omega(x, Rotation::next())` returns
   $\omega \cdot x$. Why does it not need to know which polynomial
   is being rotated? (One sentence answer; the reason is in the
   structure of evaluation at $\omega^j x$.)

### Answers in the code

- Exercise 1: `F::S = 32` for both Pallas and Vesta; see
  [`pasta_curves::fields`](https://github.com/zcash/pasta_curves)
  or the `ROOT_OF_UNITY` constants.
- Exercise 2: the operations are (i) zero-pad to length $n'$, (ii)
  multiply by powers of $\zeta$, (iii) extended FFT, (iv) extended
  IFFT and divide out the coset shift.

## 7. Further reading

- Sean Bowe's
  [original announcement of Halo](https://electriccoin.co/blog/explaining-halo-2/)
  explains why the extended domain matters for the proof system.
