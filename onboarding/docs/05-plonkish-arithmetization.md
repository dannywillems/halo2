---
sidebar_position: 5
title: The PLONKish Arithmetization
description: Columns, selectors, expressions, gates, and the ConstraintSystem object that ties them together.
---

# The PLONKish Arithmetization

## 1. Why this chapter exists

Before any commitment, any FFT, any polynomial, halo2 starts with
an *arithmetization*: a rectangular table of cells, a fixed list of
columns, and a fixed list of low-degree polynomial constraints. The
constraint system is the user-facing API. Almost every halo2 chip
boils down to "allocate a few columns, write a few constraints
relating them". If you understand chapter 04 (polynomial domains)
and this chapter, you can read the prover in chapter 11 with no
surprises. This is the most important chapter in the course.

## 2. Definitions

Fix the small domain $H = \{1, \omega, \dots, \omega^{n-1}\}$ from
chapter 04, with $n = 2^k$.

**Definition (PLONKish table).** A table of $n$ rows and
$N = N_a + N_f + N_i$ columns, partitioned into:

- $N_a$ *advice columns* (witness data),
- $N_f$ *fixed columns* (selectors and lookup tables),
- $N_i$ *instance columns* (public inputs).

Each column is a function $A : H \to \mathbb{F}_p$ (we identify
the row $i$ with $\omega^i$).

**Definition (Rotation).** A rotation $r \in \mathbb{Z}$ acts on a
column $A$ by $(A \cdot \omega^r)(\omega^i) = A(\omega^{i+r})$. In
the Lagrange basis this is the index shift by $r \bmod n$. The
relative rotations a gate may use are bounded: the verifier must
preimage every distinct $(\text{column}, r)$ pair in the proof.

**Definition (Expression).** A formal arithmetic combination over
constants, column queries, and selectors. The grammar:

$$
\begin{aligned}
E ::=&\ \mathsf{Constant}(c) \\
   |&\ \mathsf{Selector}(s) \\
   |&\ \mathsf{Fixed}(A, r)\ |\ \mathsf{Advice}(A, r)\ |\ \mathsf{Instance}(A, r) \\
   |&\ -E\ |\ E + E\ |\ E \cdot E\ |\ \alpha \cdot E
\end{aligned}
$$

**Definition (Custom gate).** A list of expressions $(p_1, \dots,
p_g)$ supplied to `ConstraintSystem::create_gate`. The proving system
enforces $p_j(\omega^i) = 0$ for every row $i$ and every gate index
$j$; that is, $t_H \mid p_j$ for each $j$.

**Definition (Simple selector).** A selector $s$ promised by the
caller to multiply at most one other selector at a time. Simple
selectors can be combined into fewer fixed columns by the
selector-combining optimization; see `dev/util.rs` and the design
chapter referenced below.

**Definition (Permutation argument).** Equality constraints
between cells (`Region::constrain_equal`) are realized by a
permutation argument over the chosen *equality columns*; see
chapter 08.

**Definition (Lookup argument).** A claim that the values in some
columns at every row appear in a fixed table; see chapter 07.

## 3. The code

### 3.1 The four column types

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L18-L86)

Tag types `Advice`, `Fixed`, `Instance` are zero-sized markers;
`Any` is the runtime enum. `Column<C>` is `(index, marker)`.

### 3.2 Selectors

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L255-L269)

A `Selector(idx, is_simple)` is a virtual fixed column that takes
values in $\{0, 1\}$. Two flavours exist:

- *Simple selectors* (constructed by `meta.selector()`): may be
  combined together at keygen time to save columns.
- *Complex selectors* (constructed by `meta.complex_selector()`):
  always materialize as their own fixed column. Required whenever
  the selector is multiplied by another selector inside a gate or
  used inside a lookup expression.

### 3.3 Queries

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L272-L302)

Every distinct `(column, rotation)` pair becomes one entry in the
proof's evaluation list. Reusing the same query in many gates is
free; introducing a new rotation costs one extra opening.

### 3.4 The `TableColumn` type

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L304-L336)

`TableColumn` wraps a `Column<Fixed>` for use inside lookup
arguments. The wrapper exists to prevent chip authors from
accidentally loading a lookup table into a column without
default-padding the unused rows, which would create a soundness
bug. The documentation comment on `inner` spells this out
explicitly.

### 3.5 The `Circuit` trait

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L466-L486)

The two phases:

- `configure(meta)`: pure shape declaration. The implementer
  allocates columns, declares gates, declares lookups, returns the
  `Config`. No witness data is available here.
- `synthesize(config, layouter)`: assignment phase. The implementer
  walks the circuit and writes witness values into the cells via
  the `Layouter` (see chapter 06).

The third method `without_witnesses` returns a witness-less copy
of `self`; it is what keygen and the verifier construct so they can
call `configure` and discover the shape without ever seeing the
witness.

### 3.6 `Expression`

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L488-L510)

`Expression<F>` is a tree of column queries, selector references,
constants, and arithmetic combinators. Its core operation is
`evaluate`, a fold parameterized by callbacks for each variant;
this is how the same expression is reused by the prover (evaluated
on the extended coset), the verifier (evaluated at a single point),
and the dev tools (evaluated row by row).

### 3.7 `Constraints::with_selector`

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L851-L875)

`Constraints::with_selector(s, iter)` multiplies every constraint
in `iter` by the selector expression `s`. This is the idiom for
"these constraints are only enforced where the selector is 1". Use
it instead of manually multiplying by `s` inside each expression:
the helper handles the iterator-of-tuple conversion and keeps the
named-constraint debug labels intact.

### 3.8 `ConstraintSystem`

**Source:** [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs#L932-L965)

The bag of state that `configure` builds up. The interesting
fields:

- `num_advice_columns`, `num_fixed_columns`, `num_instance_columns`,
  `num_selectors`: the shape counts.
- `gates`: the list of custom gates.
- `advice_queries`, `instance_queries`, `fixed_queries`: deduplicated
  `(column, rotation)` lookups. Their length feeds the proof size.
- `permutation`: the permutation argument's accumulator of columns
  that must support equality constraints.
- `lookups`: the list of lookup arguments.
- `constants`: a small set of fixed columns reserved for the
  constant-loading workflow.
- `minimum_degree`: a contributor-overridable lower bound on the
  global degree, used when the user wants to leave room for future
  growth without rebuilding the keys.

### 3.9 A minimal worked example

The canonical example circuit lives in
[`halo2_proofs/examples/simple-example.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/simple-example.rs).
The multiplication gate is declared inside the `configure` block:

**Source:** [`halo2_proofs/examples/simple-example.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/simple-example.rs#L60-L115)

The constraint `s_mul * (lhs * rhs - out) = 0` becomes a single
`Expression<F>` evaluated row by row.

## 4. Failure modes

- **Simple selector misuse.** Multiplying two simple selectors
  inside the same gate breaks the selector-combining optimization
  and produces a wrong proving key. Use `complex_selector()` if
  you need this.
- **Forgetting to call `meta.enable_equality(col)`.** Equality
  constraints (`region.constrain_equal`) only work on columns the
  permutation argument was told about. The error surface is
  `Error::ColumnNotInPermutation`.
- **High-degree gates.** The gate degree directly drives
  `extended_k` in chapter 04. A degree-9 gate roughly doubles the
  extended domain size relative to a degree-5 gate; that doubles
  prover time and proof size.
- **Lookup encumbrance.** A `TableColumn` is exclusive to its
  lookup; you cannot also use the underlying `Column<Fixed>` as a
  general-purpose fixed column. Trying to results in
  `Error::ColumnNotInPermutation` or a `MockProver` failure.

## 5. Spec pointers

- [PLONK paper, eprint 2019/953](https://eprint.iacr.org/2019/953),
  section 4: the standard PLONK arithmetization. The halo2
  arithmetization generalizes this by allowing custom gates and
  more advice columns.
- The
  [Halo 2 Book "PLONKish Arithmetization" chapter](https://zcash.github.io/halo2/concepts/arithmetization.html)
  covers the same material from a user's perspective.
- The
  [Halo 2 Book "Selector combining" chapter](https://zcash.github.io/halo2/design/implementation/selector-combining.html)
  explains the simple-vs-complex selector distinction and the
  packing algorithm that justifies it.

## 6. Exercises

1. Open
   [`halo2_proofs/examples/simple-example.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/simple-example.rs)
   and identify the columns the example declares. How many advice,
   fixed, instance columns? Which one is the multiplication
   selector?
2. Extend the example with a second gate `add` that enforces
   `s_add * (lhs + rhs - out) = 0` on the same advice columns. Run
   `cargo run --release --example simple-example` to check the
   proof still verifies.
3. Find the assertion in
   [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs)
   that forbids two simple selectors from multiplying each other
   inside a gate. (Hint: search for "simple selector".) State in
   one sentence what the constraint is.
4. Why does `Circuit::without_witnesses` exist? Answer in one
   sentence by reading the doc comment in
   [`halo2_proofs/src/plonk/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/circuit.rs).

### Answers in the code

- Exercise 1: two advice columns `[a0, a1]`, one instance column,
  one fixed column for the constant, one selector `s_mul`. See
  the `FieldConfig` struct in
  [`simple-example.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/simple-example.rs).
- Exercise 3: the check is in `ConstraintSystem::create_gate`; the
  error message mentions "Simple selectors are not allowed to be
  multiplied by other simple selectors".
- Exercise 4: keygen and the verifier need to call `configure`
  *without* knowing the witness; `without_witnesses` returns a
  copy with all `Value`s set to `None`.

## 7. Further reading

- Daira Hopwood's
  [PLONKish arithmetization explainer](https://github.com/daira/r1cs-vs-plonkish).
- The "Custom gates" section of the
  [Halo 2 Book](https://zcash.github.io/halo2/user/simple-example.html).
