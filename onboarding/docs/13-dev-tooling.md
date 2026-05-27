---
sidebar_position: 13
title: Dev Tooling -  MockProver and friends
description: MockProver, CircuitCost, CircuitGates, TracingFloorPlanner, and the dev-graph layout renderer.
---

# Dev Tooling: MockProver and friends

## 1. Why this chapter exists

Writing a halo2 circuit without `MockProver` is theatre. Every
gadget chapter in this course assumes you can run `MockProver`
against your code and see a `VerifyFailure` that names the row
and constraint at fault. This chapter is the catalogue of the
dev module: what it does, what it does *not* do (it is not a
substitute for `verify_proof`), and which dev tool to reach for
in each debugging situation.

## 2. Definitions

**Definition (Mock proof).** A row-by-row evaluation of every
gate constraint, lookup, and permutation, performed *without* any
commitment or Fiat-Shamir. A passing mock proof means the
witness is consistent with the constraints; it does not prove
that `create_proof` will produce a valid proof, because it does
not exercise the polynomial commitment scheme. Conversely, a
passing `verify_proof` does not imply that `MockProver` would
also pass, because `MockProver` is stricter about row-level
identities (a polynomial that vanishes on $H$ but is not
identically zero passes `verify_proof` and fails `MockProver`).

**Definition (Verify failure).** A typed error reported by
`MockProver::verify` that names exactly the gate / lookup /
permutation constraint and the row that failed. The full set is
in
[`halo2_proofs/src/dev/failure.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/failure.rs).

**Definition (Circuit cost).** A static estimate of the
proof-time MSM cost, FFT cost, and proof size for a given
circuit shape. Useful when comparing two ways of expressing the
same constraint.

## 3. The code

### 3.1 The dev module surface

**Source:** [`halo2_proofs/src/dev.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev.rs#L1-L42)

Public exports:

- `MockProver`
- `VerifyFailure`, `FailureLocation`
- `CircuitCost`
- `CircuitGates`
- `TracingFloorPlanner`
- `CircuitLayout` and `circuit_dot_graph` (gated by feature
  `dev-graph`).

### 3.2 `MockProver`

**Source:** [`halo2_proofs/src/dev.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev.rs#L270-L300)

Two entry points:

- `MockProver::run(k, &circuit, instances)`: synthesize and
  return the prover state.
- `MockProver::verify()`: check every constraint and return
  `Ok(())` or a vector of `VerifyFailure`s.

The convenience method `assert_satisfied` panics on the first
failure with a pretty-printed message. Use this in tests; use
`verify` when you want to programmatically inspect failures.

A worked example, lifted from the doc comment of `run`:

```rust
let circuit = MyCircuit { ... };
MockProver::<Fp>::run(k, &circuit, vec![public_inputs])?
    .assert_satisfied();
```

### 3.3 `CircuitCost`

**Source:** [`halo2_proofs/src/dev/cost.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/cost.rs#L20-L80)

`CircuitCost::measure(k, &circuit)` returns a struct exposing
the marginal cost per proof (advice columns, fixed columns,
gates, lookup arguments, equality constraints, etc.). The
`marginal_proof_size` method returns a byte count estimate.

### 3.4 `CircuitGates`

**Source:** [`halo2_proofs/src/dev/gates.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/gates.rs#L80-L150)

`CircuitGates::collect::<MyCircuit>()` walks the configure step
and prints every gate's name, degree, and column references.
Useful when chasing "why is my circuit's max degree 9 instead of
5".

### 3.5 `TracingFloorPlanner`

**Source:** [`halo2_proofs/src/dev/tfp.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/tfp.rs#L60-L120)

Wraps any floor planner and emits `tracing` events for each
region, table, and namespace push. Use it from a test by setting
`Circuit::FloorPlanner = TracingFloorPlanner<V1>` and turning on
`tracing_subscriber` at info level.

### 3.6 The dev-graph layout

**Source:** [`halo2_proofs/src/dev/graph/layout.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/graph/layout.rs#L30-L100)

`CircuitLayout::default().render(k, &circuit, &drawing)` renders
the column-by-row occupancy as a PNG via `plotters`. The
canonical use is
[`halo2_proofs/examples/circuit-layout.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/circuit-layout.rs).
Compile with `--features dev-graph` (and `test-dev-graph` when
running tests).

## 4. Failure modes

- **Trusting `MockProver::verify` as a proof.** It is not a
  proof. It is a faster, more diagnostic check of the same row
  identities, but it skips every commitment. A successful
  `MockProver` is necessary, not sufficient.
- **Forgetting `--features dev-graph`.** `CircuitLayout::render`
  is gated behind this feature. Building it without the feature
  is a no-op.
- **`assert_satisfied` in production code.** It panics; only call
  it inside `#[test]`.
- **Reading `CircuitCost::marginal_proof_size` as the actual
  proof byte count.** It is a static estimate that ignores
  feature-flag-driven differences. Always cross-check with a
  real proof.

## 5. Spec pointers

- The
  [Halo 2 Book "Developer tools" chapter](https://zcash.github.io/halo2/user/dev-tools.html)
  walks through the same APIs from a user's perspective.

## 6. Exercises

1. Run
   `cargo test --release --features dev-graph -p halo2_proofs --example circuit-layout`
   and inspect the resulting PNG. Identify which region in
   `circuit-layout.rs` corresponds to which colored rectangle in
   the image.
2. Add a `CircuitCost` measurement to `simple-example.rs` and
   print the marginal proof size. Compare against the actual
   length of the proof bytes after `create_proof`.
3. Take a known-buggy circuit (intentionally underconstrained)
   and run `MockProver`. Confirm the `VerifyFailure` reports the
   correct row and constraint name.
4. Switch the `simple-example` to use `TracingFloorPlanner<V1>`
   and run with `RUST_LOG=trace`. Identify the region annotations
   the tracing output emits.

### Answers in the code

- Exercise 1: each `assign_region` call in `circuit-layout.rs`
  produces one labeled rectangle. The labels are the strings
  passed to the region.

## 7. Further reading

- The
  [`tracing`](https://docs.rs/tracing) crate's introductory
  guide, since `TracingFloorPlanner` builds on it.
