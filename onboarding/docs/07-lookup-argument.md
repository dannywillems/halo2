---
sidebar_position: 7
title: The Lookup Argument
description: The Plookup-style lookup argument as implemented in halo2_proofs.
---

# The Lookup Argument

## 1. Why this chapter exists

Custom gates can express polynomial relations of bounded degree.
Many useful relations (range checks, bit decomposition, AND / XOR
tables, S-boxes) are easier to express as "this value is one of
the rows of this table". The lookup argument lets a circuit assert
exactly that. It is the second-most-used soundness mechanism in
halo2 after the gate system itself, and it is the one most often
misunderstood. This chapter covers the math and the code so that
you can write your own lookup chip and reason about the proof-size
cost.

## 2. Definitions

Fix $n = 2^k$ and the small domain $H$ from chapter 04.

**Definition (Multiset equality).** Two multisets $A$ and $S$ of
size $n$ are equal as multisets iff their characteristic
polynomials agree:
$$\prod_{i=0}^{n-1} (X - A_i) = \prod_{i=0}^{n-1} (X - S_i).$$

**Definition (Plookup argument).** Given an input multiset
$A = (a_0, \dots, a_{n-1})$ and a table multiset
$S = (s_0, \dots, s_{n-1})$, the prover wants to convince the
verifier that $A \subseteq S$ as a multiset. Plookup reduces this
to a multiset-equality check on the sorted concatenation
$(A, S)$ vs a specific permutation; the resulting check is
expressed as a degree-bounded grand product. See
[eprint 2020/315](https://eprint.iacr.org/2020/315) for the full
construction.

**Definition (Multi-column lookup).** When the lookup has $m$
input expressions and $m$ table expressions, the prover uses a
verifier-supplied challenge $\theta$ to collapse them into one
column: $\tilde a = \sum_{j=0}^{m-1} \theta^{m-1-j} a_j$ and
similarly for $\tilde s$. The single-column Plookup argument is
then run on $\tilde a$ and $\tilde s$.

**Definition (Grand product polynomial $z$).** The accumulator
polynomial of the lookup argument; the prover commits to $z$ and
the verifier checks four boundary and recurrence relations on it
(see the comment block in
[`plonk/lookup.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup.rs#L25-L57)).

## 3. The code

### 3.1 The argument struct

**Source:** [`halo2_proofs/src/plonk/lookup.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup.rs)

`Argument` is the parsed declaration of one lookup: a pair of
`Vec<Expression<F>>` of equal length, one for inputs, one for
table cells. The `required_degree` method records, in code, the
exact identities the verifier will check; it is the cleanest place
in the codebase to read the Plookup algebra.

### 3.2 Declaring a lookup

`ConstraintSystem::lookup` is the public API:

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L1057-L1085)

The closure receives a `VirtualCells` (so it can call
`meta.query_advice` etc.) and returns a vector of
`(input_expression, table_expression)` pairs. The table expression
must reference a `TableColumn`, never a plain `Column<Fixed>`. This
is enforced by the types (`query_lookup_table` returns an
`Expression<F>` that captures the table-column origin).

### 3.3 The prover

**Source:** [`halo2_proofs/src/plonk/lookup/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup/prover.rs#L1-L60)

The prover steps, in order:

1. Compute $\tilde a$ and $\tilde s$ in `LagrangeCoeff`,
   absorbing the challenge $\theta$.
2. Sort $\tilde a$ into $a'$ and the matching slice of $\tilde s$
   into $s'$ such that $a'$ is non-decreasing relative to $s'$
   (the "compressed permutation").
3. Commit to $a'$ and $s'$.
4. Squeeze $\beta$ and $\gamma$ from the transcript.
5. Compute the grand product $z$ with $z(\omega^0) = 1$ and
   $z(\omega^{i+1}) = z(\omega^i) \cdot \frac{(a_i + \beta)(s_i + \gamma)}{(a'_i + \beta)(s'_i + \gamma)}$.
6. Commit to $z$.

### 3.4 The verifier

**Source:** [`halo2_proofs/src/plonk/lookup/verifier.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup/verifier.rs)

The verifier checks four identities at the challenge point $x$:

- $L_0(x) \cdot (1 - z(x)) = 0$ (boundary at $\omega^0$),
- $L_{\text{last}}(x) \cdot (z(x)^2 - z(x)) = 0$ (boundary at
  $\omega^{n-1}$),
- the main grand-product recurrence (the long expression from the
  comment in
  [`plonk/lookup.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup.rs#L25-L57)),
- the two side constraints that pin $a'$ and $s'$ to the right
  permutation of $A$ and $S$.

### 3.5 The dev tool's view

`MockProver` runs all four identities row by row in `LagrangeCoeff`
without any commitment. If a circuit's lookup fails, `MockProver`
reports a `VerifyFailure::Lookup { lookup_index, location }`
(defined in
[`halo2_proofs/src/dev/failure.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/failure.rs))
naming the offending row and the lookup index in declaration order.

## 4. Failure modes

- **Unpadded table column.** A lookup table column with unfilled
  rows defaults its remaining cells to zero. If zero is not in the
  table semantically, the prover will fail to produce a valid $z$
  for any row whose input is zero. The
  `Layouter::assign_table` API forces you to fill every row, but
  it is easy to bypass.
- **Reusing a `TableColumn` for something else.** A column wrapped
  in `TableColumn` is encumbered for the lookup; trying to use it
  as a plain fixed column raises an error at keygen.
- **Forgetting that a complex selector is required.** The lookup
  expression must not multiply two simple selectors. If you want
  to gate a lookup on a selector, use `meta.complex_selector()`.
- **Order of `(input, table)` pairs.** The pairs in the closure
  passed to `meta.lookup(...)` define the columns of the
  multi-column lookup. Permuting them changes the challenge
  $\theta$'s effect; you cannot reorder them without changing the
  verifying key.

## 5. Spec pointers

- [Gabizon and Williamson, "plookup",
  eprint 2020/315](https://eprint.iacr.org/2020/315).
- The
  [Halo 2 Book "Lookup argument" chapter](https://zcash.github.io/halo2/design/proving-system/lookup.html)
  reproduces the algebra of the comment block above with worked
  examples.
- The
  [Halo 2 Book "Lookup tables" chapter](https://zcash.github.io/halo2/user/lookup-tables.html)
  is a user-side walk-through.

## 6. Exercises

1. Open
   [`halo2_proofs/src/plonk/lookup.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/lookup.rs)
   and read the doc comment of `required_degree`. Write down the
   four identities it enumerates in your own words.
2. Read the lookup chapter of the Halo 2 Book and identify each
   of the four identities in section 3.4 above with a numbered
   line in the comment block.
3. Open the SHA-256 table16 chip:
   [`halo2_gadgets/src/sha256/table16/spread_table.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sha256/table16/spread_table.rs).
   Find the `meta.lookup(...)` call. What does the table contain
   and how many rows?
4. Trigger a lookup failure in `MockProver`. Take any test that
   uses `lookup_range_check` and modify the witness so that a
   value is out of range. Run the test and inspect the
   `VerifyFailure::Lookup` message.

### Answers in the code

- Exercise 1: the four identities are (i) $z(\omega^0) = 1$, (ii)
  $z(\omega^{n-1}) \in \{0, 1\}$, (iii) the grand-product
  recurrence, (iv) the boundary and step constraints on
  $(a', s')$. All four are in the same comment.
- Exercise 3: the spread table maps 16-bit dense values to their
  "spread" form (bits interleaved with zeros); it has $2^{16}$
  rows.

## 7. Further reading

- The original [PLONK + plookup blog post](https://research.protocol.ai/posts/plookup/).
- Section "Plookup" of the
  [Halo 2 design book](https://zcash.github.io/halo2/design.html).
