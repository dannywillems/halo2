---
sidebar_position: 6
title: Circuit Synthesis -  Chip, Region, FloorPlanner
description: The assignment-phase API a circuit author actually uses, and how SimpleFloorPlanner and V1 differ.
---

# Circuit Synthesis: Chip, Region, FloorPlanner

## 1. Why this chapter exists

Chapter 05 covered what a circuit *declares*. This chapter covers
what a circuit *does*: how synthesis walks the table, assigns
values, and produces the witness consumed by the prover. The
chip-and-region API is what every gadget author writes against, and
it carries a few invariants that are not obvious from the type
signatures. Get these wrong and your circuit will be either
unsound or unprovable.

## 2. Definitions

**Definition (Region).** A contiguous, opaque block of rows owned
by a single chip. Inside a region, the chip uses *relative* offsets
($0, 1, 2, \dots$); the layouter decides the absolute starting
row. Cells inside a region can be freely cross-referenced; cells
across regions must be tied together with an explicit equality
constraint.

**Definition (Cell).** A pointer
$(\text{region index}, \text{row offset}, \text{column})$ into
the table. The cell does not own a value; it identifies a
location.

**Definition (AssignedCell).** A `(Value, Cell)` pair. The
`Value` is the witness data (in `Some(_)` mode during proving, in
`None` mode during keygen and verification).

**Definition (Layouter).** The object that drives the
`synthesize` method. The layouter exposes a single primitive,
`assign_region`, which takes a closure that operates on a `Region`.
The layouter is responsible for placing regions in the table.

**Definition (Floor planner).** The strategy the layouter uses to
place regions. Two are shipped:

- `SimpleFloorPlanner`: places regions back to back, one after
  another, in the order they are declared. Predictable, but
  wasteful: a region that uses few columns leaves the others
  empty.
- `V1` (first-pass-second-pass planner): measures the shapes of
  every region in a first pass, then places them by packing
  columns. Substantially denser layouts, but the planner runs the
  synthesis closure twice, so closures must be deterministic and
  must not depend on side effects.

## 3. The code

### 3.1 The `Chip` trait

**Source:** [`halo2_proofs/src/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit.rs#L20-L49)

`Chip::Config` is the type returned by your chip's `configure`
function and consumed by `synthesize`. `Chip::Loaded` is for chip
state loaded at the start of synthesis (e.g. a precomputed lookup
table); use `()` if you have none.

### 3.2 `Cell` and `AssignedCell`

**Source:** [`halo2_proofs/src/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit.rs#L86-L183)

The two layered types:

- `Cell`: pure pointer.
- `AssignedCell<V, F>`: pointer + value. The value type `V` is
  often `F` for plain witness data, but `Assigned<F>` for cells
  that participate in lazy inversions (see
  [`plonk/assigned.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk/assigned.rs)).

`AssignedCell::copy_advice` is the canonical way to forward a
witness value from one region into another: it assigns the value
into the new cell *and* issues the equality constraint.

### 3.3 The `Region` API

**Source:** [`halo2_proofs/src/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit.rs#L185-L210)

The methods you will use most:

- `assign_advice(annotation, column, offset, value)`: write to an
  advice cell.
- `assign_fixed(annotation, column, offset, value)`: write to a
  fixed cell.
- `assign_advice_from_instance(...)`: write to advice and
  constrain it equal to a cell in an instance column.
- `assign_advice_from_constant(...)`: write to advice and
  constrain it equal to a constant (uses one of the
  `ConstraintSystem::constants` columns).
- `constrain_equal(left, right)`: add an equality constraint
  between two cells. Both columns must be in the permutation.
- `enable_selector(annotation, selector, offset)`: turn on a
  selector at a given row in the region.

### 3.4 The `Layouter` trait

**Source:** [`halo2_proofs/src/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit.rs#L420-L495)

Most of the time, a chip only calls:

- `layouter.assign_region(name, closure)`
- `layouter.assign_table(name, closure)` (for filling lookup
  tables; uses `Table` instead of `Region`).
- `layouter.constrain_instance(cell, column, row)` (to expose a
  cell to a public-input row).
- `layouter.namespace(name)`: a debug-only annotation that nests
  region names for traces and `dev` output. Returns a
  `NamespacedLayouter` that flushes the namespace on `Drop`.

### 3.5 `SimpleFloorPlanner`

**Source:** [`halo2_proofs/src/circuit/floor_planner/single_pass.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit/floor_planner/single_pass.rs#L20-L39)

Single-pass. Each region starts on the row after the previous
region ended, regardless of column usage. The closure is invoked
once. Use this for clarity in tests and for any circuit small
enough that layout density does not matter.

### 3.6 `V1` floor planner

**Source:** [`halo2_proofs/src/circuit/floor_planner/v1.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit/floor_planner/v1.rs#L20-L40)

Two passes:

- *Measurement pass*: a fake layouter (`V1Pass` in measurement
  mode) records the *shape* of each region (which columns it
  uses, how many rows). No witness values are written.
- *Assignment pass*: a solver assigns each region a starting row
  based on the measured shapes, then the closures are invoked a
  second time and this time the values are actually written.

This is why your `assign_region` closure must be a pure function of
its inputs: it will run twice. Capturing a `Cell::new()` counter or
a file handle inside the closure is wrong.

The 0.3.0 release contains the cautionary tale: the V1 planner
used to depend on `slice::sort_unstable_by_key`, which is
deterministic on a given target but produces different orderings
across 32-bit and 64-bit targets. The fix in
[`halo2_proofs/CHANGELOG.md`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/CHANGELOG.md)
was to switch to a stable sort, and to gate the old behaviour
behind `floor-planner-v1-legacy-pdqsort` to keep the Orchard
Verifying Key bit-identical.

## 4. Failure modes

- **Side-effecting region closures.** Anything stored in a
  `Cell::new()` outside the closure will be observed twice by `V1`.
  The fix is to capture only values that are computed
  deterministically from the closure arguments.
- **Cross-region cells without equality.** Forgetting
  `constrain_equal` (or, equivalently, `copy_advice`) means the two
  cells are independent witness slots; the witness can be made
  consistent locally but the constraint is missing globally. This
  is a soundness bug.
- **`assign_advice_from_constant` without `enable_constant`.** The
  constant-loading workflow requires
  `meta.enable_constant(column)` in `configure`. Without it the
  layouter has no fixed column to store the constant in, and
  synthesis returns `Error::NotEnoughColumnsForConstants`.
- **Mixing `SimpleFloorPlanner` and `V1` between keygen and prove.**
  The floor planner is part of the verifying key. If keygen used
  `V1` and proving uses `SimpleFloorPlanner`, the column
  assignments will not match the committed fixed columns. The
  `Circuit::FloorPlanner` associated type exists precisely so this
  cannot happen accidentally.

## 5. Spec pointers

- The
  [Halo 2 Book "Floor plan" chapter](https://zcash.github.io/halo2/concepts/chips.html)
  introduces the chip / region model.
- The
  [V1 floor planner blog post](https://electriccoin.co/blog/halo2-floor-planner/)
  by Jack Grigg walks through the measurement / assignment two-pass
  algorithm.

## 6. Exercises

1. Open the `two-chip` example
   [`halo2_proofs/examples/two-chip.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/two-chip.rs).
   Identify (a) which chip allocates each column, (b) which cells
   are passed between chips and how (`AssignedCell` returned from
   one and consumed by the next).
2. Change `simple-example.rs` from `SimpleFloorPlanner` to `V1`
   (the type alias is `halo2_proofs::circuit::floor_planner::V1`)
   and re-run. Does the proof still verify?
3. Modify `two-chip.rs` so that the multiplication chip writes its
   output to a *new* region and intentionally omits the
   `constrain_equal` that ties it to the consumer. Run the example
   under `MockProver::verify` (replace the proof code with a
   `MockProver::run`) and observe the `VerifyFailure` that the
   dev tool reports.
4. The doc comment on `Region` mentions a `TODO` about "logical
   columns guaranteed to correspond to the chip". Find it in
   [`halo2_proofs/src/circuit.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/circuit.rs)
   and propose, in two sentences, what a fix could look like.

### Answers in the code

- Exercise 1: the `FieldChip` allocates the advice and the
  multiplication selector; the `AddChip` allocates the addition
  selector; the chips share advice columns and pass `AssignedCell`s
  between them.
- Exercise 3: the resulting `VerifyFailure` will be
  `Permutation { column, location }` for the location of the
  omitted equality.

## 7. Further reading

- The dev tool `TracingFloorPlanner` in
  [`halo2_proofs/src/dev/tfp.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/dev/tfp.rs)
  wraps any floor planner and logs each region assignment via the
  `tracing` crate. Wrap your circuit in it once if you ever wonder
  why layout looks the way it does.
