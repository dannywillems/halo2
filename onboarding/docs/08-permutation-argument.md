---
sidebar_position: 8
title: The Permutation Argument
description: How equality constraints between cells become a grand-product check.
---

# The Permutation Argument

## 1. Why this chapter exists

Any time you write `region.constrain_equal(a, b)` or
`AssignedCell::copy_advice(...)` you are using the permutation
argument. It is the mechanism that lets a circuit author treat the
table as a graph of cells with edges, rather than a flat collection
of rows. The argument is a routine generalization of Plonk's
copy-constraint argument, but the multi-column and chunking
machinery is non-obvious. If you ever change anything in
`plonk/permutation/`, you need to know what each polynomial in the
proof represents.

## 2. Definitions

Fix $n = 2^k$, $H = \{1, \omega, \dots, \omega^{n-1}\}$.

**Definition (Wire indexing).** Suppose the columns enrolled in
the permutation argument are
$P_0, \dots, P_{c-1}$ (each a column of the table). The set of
*wires* is the product $\{0, 1, \dots, c-1\} \times H$ of column
indices and row labels, giving $c \cdot n$ wires total.

**Definition (Permutation $\sigma$).** A permutation
$\sigma : \{0, \dots, c-1\} \times H \to \{0, \dots, c-1\} \times H$.
The cycles of $\sigma$ are the equivalence classes of cells joined
by `constrain_equal`. The chunking required when $c$ is large is
described below.

**Definition (Cycle-product identity).** Define the wire identity
$P(j, \omega^i) = (\text{value at column } j, \text{row } i)$. The
permutation $\sigma$ is consistent with the witness iff, for every
pair of $(\beta, \gamma) \in \mathbb{F}_p^2$,
$$
\prod_{j, i} (P(j, \omega^i) + \beta \cdot \delta^j \cdot \omega^i + \gamma)
\;=\;
\prod_{j, i} (P(j, \omega^i) + \beta \cdot s_{\sigma}(j, \omega^i) + \gamma)
$$
where $\delta$ is a fixed coset shift (a quadratic non-residue) and
$s_\sigma$ is a polynomial encoding of $\sigma$. The argument is
encoded by committing to a running product polynomial $z$ that
realizes this identity.

**Definition (Chunking).** With many columns, the polynomial $z$
would have unbounded degree. The argument splits the columns into
*chunks* of size at most $\text{degree}(z) - 1$ and accumulates a
separate grand product $z_0, z_1, \dots$ per chunk. Chunk boundaries
introduce additional boundary identities, which is why the comment
block in
[`plonk/permutation.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/permutation.rs)
lists "intermediate" and "final" boundary constraints separately.

## 3. The code

### 3.1 The argument struct

```rust file=../../halo2_proofs/src/plonk/permutation.rs title="halo2_proofs/src/plonk/permutation.rs"
```
`Argument` carries the list of columns enrolled in the permutation;
`required_degree` records the four constraint patterns the verifier
checks (in the same style as the lookup chapter):

- $L_0(X) \cdot (1 - z(X)) = 0$,
- a chunk-internal grand-product recurrence over
  $\prod_j (p_j(X) + \beta s_j(X) + \gamma) / \prod_j (p_j(X) + \beta \delta^j X + \gamma)$,
- a "chain" constraint $L_0(X) \cdot (z_k(X) - z_{k-1}(\omega^{n-1} X)) = 0$
  for $k \geq 1$, linking chunk $k$ to the end of chunk $k - 1$,
- $L_{\text{last}}(X) \cdot (z_\text{last}(X)^2 - z_\text{last}(X)) = 0$,
  ensuring the last value is in $\{0, 1\}$.

### 3.2 Enrolling columns

The user-facing API:

```rust file=../../halo2_proofs/src/plonk/circuit.rs#L1050-L1060 title="halo2_proofs/src/plonk/circuit.rs"
```
`meta.enable_equality(column)` is the only entry point. It adds
the column to the permutation argument's column list, after which
`Region::constrain_equal` can target cells in that column. The
order of `enable_equality` calls matters: it determines the wire
indexing $j$, which appears in the $\delta^j$ term of the
constraint and therefore in the verifying key. Reordering it breaks
verifying-key compatibility.

### 3.3 Keygen

```rust file=../../halo2_proofs/src/plonk/permutation/keygen.rs title="halo2_proofs/src/plonk/permutation/keygen.rs"
```
The keygen step:

1. Walks the equality-constraint graph produced by the layouter,
   computes the union-find of cycles.
2. Encodes each cycle as a permutation $\sigma$ on the
   $(c \cdot n)$-wire space.
3. Builds the $c$ permutation polynomials $s_0, \dots, s_{c-1}$ in
   Lagrange basis, where $s_j(\omega^i)$ holds the next wire
   identity in $\sigma$'s cycle starting from $(j, \omega^i)$.
4. Commits to each $s_j$. The commitments become the permutation
   `VerifyingKey`.

### 3.4 The prover

```rust file=../../halo2_proofs/src/plonk/permutation/prover.rs title="halo2_proofs/src/plonk/permutation/prover.rs"
```
The prover (per chunk):

1. Computes the numerator product $\prod_j (P_j(X) + \beta \delta^j X + \gamma)$ and the denominator
   $\prod_j (P_j(X) + \beta s_j(X) + \gamma)$ on the small domain.
2. Inverts the denominator using a batch inversion.
3. Walks the rows building $z_k$ as the running product.
4. Commits to $z_k$.

### 3.5 The verifier

```rust file=../../halo2_proofs/src/plonk/permutation/verifier.rs title="halo2_proofs/src/plonk/permutation/verifier.rs"
```
The verifier expects to find one commitment per chunk in the
transcript, queries each $z_k$ at $x$ and $\omega \cdot x$, then
checks the four identities listed in 3.1 above. The check at
$L_{\text{last}}$ that $z_\text{last} \in \{0, 1\}$ is what closes
the loop, since the recurrence and boundary identities together
imply $z_\text{last} = 1$ exactly when the multiset equality
holds.

## 4. Failure modes

- **Forgetting `enable_equality`.** The most frequent
  contributor mistake. The error is
  `Error::ColumnNotInPermutation`; the fix is to add
  `meta.enable_equality(col)` in the configure block.
- **Changing the enrollment order.** Adding a new
  `enable_equality(col)` call between two existing ones shifts
  every later column's wire index by one, which changes the
  $\delta^j$ term in the proof identity and breaks every existing
  verifying key for that circuit. Add new columns at the end of
  the enrollment list.
- **Mixing chunking sizes.** The chunk size is implied by the
  circuit's degree. If you increase the gate degree, the chunk
  size grows, the number of chunks shrinks, and the verifying key
  changes. This is one of the surprising consequences of "adding
  a high-degree custom gate".
- **Inappropriate use of `assign_advice_from_constant`.** This
  path piggybacks on the permutation argument via the
  `ConstraintSystem::constants` columns. If the constants column
  was not enrolled (the user forgot to add it via
  `meta.enable_constant`), the helper returns
  `NotEnoughColumnsForConstants`.

## 5. Spec pointers

- [PLONK paper, eprint 2019/953, Section 5.1](https://eprint.iacr.org/2019/953):
  the original permutation argument.
- The
  [Halo 2 Book "Permutation argument" chapter](https://zcash.github.io/halo2/design/proving-system/permutation.html)
  works through the multi-column chunking in full.
- Tornado-style permutation arguments are surveyed in
  [Yao Sun's PLONK notes](https://www.youtube.com/watch?v=p_eXiUlF5dY)
  for a less terse exposition.

## 6. Exercises

1. Open
   [`halo2_proofs/src/plonk/permutation.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/permutation.rs)
   and read the comment in `required_degree`. Identify each of the
   four constraint patterns by name.
2. Use `MockProver` to verify a circuit, then manually mutate one
   advice cell *after* synthesis (you will need to use the dev
   tools' internal API). Confirm the failure surface is
   `Permutation { column, location }` and matches the cell you
   broke.
3. Re-read
   [`halo2_proofs/src/plonk/permutation/keygen.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/permutation/keygen.rs)
   and describe in two sentences how the cycles in the equality
   graph become the permutation $\sigma$.
4. Run `circuit-layout` and compute (by hand or with
   `CircuitCost`) how the permutation polynomial count changes when
   you add another advice column to the `simple-example` circuit.

### Answers in the code

- Exercise 1: $L_0(X)(1 - z(X))$ boundary; chunk-internal
  recurrence; chunk-chaining constraint
  $L_0(X)(z_k(X) - z_{k-1}(\omega^{n-1} X))$; final boundary
  $L_{\text{last}}(X)(z_\text{last}(X)^2 - z_\text{last}(X))$.

## 7. Further reading

- [Zcash design notes on the Halo 2 permutation argument](https://zcash.github.io/halo2/design/proving-system/permutation.html).
