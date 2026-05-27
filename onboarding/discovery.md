# Discovery Notes (Internal)

These notes are the ground truth used to derive the onboarding chapter
list. They are not deployed to the site. The course author should read
this file before adding or rewriting chapters.

## Upstream pin

- Upstream: `https://github.com/zcash/halo2`
- Fork: `https://github.com/dannywillems/halo2`
- Pinned commit (used in every source link):
  `32a87582dfb0ad9364ef3ffe71751ceab2a502ea`
- Branch at upstream: `main`

## Workspace

`Cargo.toml` declares a 4-crate workspace:

- `halo2`: thin re-export shim (`halo2_proofs::*`), version `0.1.0-beta.2`.
  Entry point is `halo2/src/lib.rs`.
- `halo2_proofs` (0.3.2): the PLONKish proving system. Public modules:
  `arithmetic`, `circuit`, `plonk`, `poly`, `transcript`, `dev`.
- `halo2_gadgets` (0.4.0): reusable chips and gadgets. Public modules:
  `ecc`, `poseidon`, `sinsemilla`, `utilities`, optional `sha256`
  (feature-gated by `unstable-sha256-gadget`).
- `halo2_poseidon` (0.1.0): standalone Poseidon hash. No `halo2_proofs`
  dependency.

Rust toolchain: `1.60.0` (channel pinned in `rust-toolchain.toml`).

## Module map (halo2_proofs)

Largest files (line count) and what they own:

- `plonk/circuit.rs` (1547): `Column`, `Selector`, `Expression`,
  `Constraint`, `Circuit` trait, `ConstraintSystem`, `VirtualCells`.
- `dev.rs` (1141): `MockProver`, public dev exports (`CircuitCost`,
  `CircuitGates`, `TracingFloorPlanner`, `CircuitLayout`).
- `plonk/prover.rs` (786): `create_proof`.
- `circuit/value.rs` (703): the `Value<F>` type and its arithmetic.
- `plonk/assigned.rs` (666): `Assigned<F>` for lazy inversions.
- `poly/evaluator.rs` (663): cached polynomial evaluation.
- `plonk/lookup/prover.rs` (647): the lookup prover.
- `poly/multiopen.rs` (628): the multiopen wrapper and query grouping.
- `poly/domain.rs` (607): `EvaluationDomain` and basis conversions.
- `circuit.rs` (580): `Chip`, `Cell`, `AssignedCell`, `Layouter`.
- `dev/failure.rs` (565): `VerifyFailure`, `FailureLocation`.

Other top-level structure:

- `arithmetic.rs`: FFT, NTT, `best_multiexp`, `parallelize`,
  re-exports from `pasta_curves::arithmetic` (`CurveAffine`,
  `CurveExt`, etc.).
- `transcript.rs`: Fiat-Shamir transcripts (`Blake2bWrite`,
  `Blake2bRead`, `Challenge255`).
- `poly/commitment.rs`: IPA `Params`, `Blind`, `MSM` re-export,
  the inner-product `create_proof` / `verify_proof`.
- `poly/multiopen/` (prover, verifier): multiopen IPA wrapper.
- `circuit/floor_planner/`: `SimpleFloorPlanner`, `V1`.
- `plonk/lookup/`: Plookup-style lookup argument.
- `plonk/permutation/`: PLONK permutation argument.
- `plonk/vanishing/`: vanishing polynomial argument.

## Module map (halo2_gadgets)

- `ecc/`: `EccInstructions`, `EccChip` (in `ecc/chip/`).
  - `chip/add.rs`, `add_incomplete.rs`: complete and incomplete
    point addition gates.
  - `chip/witness_point.rs`: witnessing curve points.
  - `chip/mul.rs`, `chip/mul_fixed/`: variable-base and fixed-base
    scalar multiplication.
- `poseidon/pow5.rs`: x^5 S-box Poseidon chip.
- `sinsemilla/chip/`: hash-to-point and generator table; the chip
  used to build MerkleCRH and SinsemillaCommit.
- `sinsemilla/merkle/`: Merkle path verification gadget.
- `sha256/table16/`: 16-bit-lookup SHA-256 implementation.
- `utilities/`: `cond_swap`, `decompose_running_sum`,
  `lookup_range_check`.

## halo2_poseidon

- `pow5.rs` is in `halo2_gadgets`; `halo2_poseidon` itself contains
  the field-only Poseidon specification (round constants, MDS,
  permutation) that the in-circuit chip references.

## Test layout

- Unit tests inline (`#[cfg(test)] mod tests` at the bottom of files).
- Integration tests: `halo2_proofs/tests/plonk_api.rs` exercises the
  full prover / verifier round trip with a fixed test vector
  (`plonk_api_proof.bin`).
- Examples (`halo2_proofs/examples/`): `simple-example.rs`,
  `two-chip.rs`, `circuit-layout.rs`, `cost-model.rs`.
- Benchmarks: `arithmetic`, `hashtocurve`, `msm`, `plonk`,
  `dev_lookup`, `fft` (Criterion harness, `cargo bench`).
- Proptest fuzzing for `plonk::Expression` evaluation and other
  algebraic invariants (search `proptest!` in
  `halo2_proofs/src/plonk/`).

## Feature flags (halo2_proofs)

- `default`: `["batch", "multicore"]`
- `multicore`: enable rayon-based parallelism.
- `dev-graph`: graphical circuit layout via `plotters` and `tabbycat`.
- `test-dev-graph`: bitmap rendering for tests.
- `gadget-traces`: backtraces in dev-mode failures.
- `sanity-checks`: extra invariant checks (slow).
- `batch`: batch verifier (`BatchVerifier`).
- `floor-planner-v1-legacy-pdqsort`: pin V1 sort to match
  pre-2023 layout (NU5 Orchard compatibility).
- `beta` / `nightly`: in-development feature gates.

## CI graph

`.github/workflows/ci.yml`:

- `test` matrix: `{stable, beta, nightly}` x
  `{ubuntu-latest, windows-latest, macOS-latest}` running
  `cargo test --release --workspace`.
- `test-32-bit`: i686 cross-compile and test (catches usize
  assumptions).
- `build`: `wasm32-unknown-unknown` and `wasm32-wasi`.
- `bitrot`: build benchmarks and examples with `--all-features`.
- `book`: `mdbook test` against `book/`.
- `codecov`: `cargo tarpaulin`.
- `doc-links`: `cargo doc` with intra-doc-link checking.
- `fmt`: `cargo fmt --all -- --check`.
- `lints-beta.yml` and `lints-stable.yml`: `clippy`.
- `bench.yml`: nightly criterion benchmarks.

## Recent activity (last 12 months)

Top-touched files (`git log --since="12 months ago" --name-only`):

- `halo2_proofs/src/poly/multiopen.rs` (5)
- `halo2_proofs/CHANGELOG.md` (5)
- `halo2_gadgets/CHANGELOG.md` (5)
- `halo2_gadgets/src/utilities/lookup_range_check.rs` (4)
- `halo2_proofs/src/poly/multiopen/prover.rs` (3)
- `halo2_proofs/Cargo.toml` (3)
- `halo2_gadgets/src/sinsemilla/merkle.rs` (2)
- `halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs` (2)
- `halo2_gadgets/src/ecc/chip/mul_fixed/short.rs` (2)

Notable security event in 2025: `0.3.1` patched a multiopen soundness
bug (duplicate `(point, commitment)` with different evaluations).
Found by Suneal at zkSecurity. The fix is in
`poly/multiopen.rs` and the surrounding modules; the change adds
`indexmap` and rejects the duplicate case in both prover and
verifier. This is a prime example to cite in the multiopen chapter.

## Issue tracker

Selected open issues at discovery time (`gh issue list --state open`):

- #810 [book] Add documentation about circuit compatibility and
  upgrades (label: `A-documentation`, `A-book`).
- #800 [book] Sinsemilla page rendering bug (label: `A-book`).
- #806 Do not include the fixed and instance column evaluations into
  the proof (research-level proposal).
- #794 `tracing` should be optional.
- #789 Modify constraint in `decompose_running_sum`.
- #803 Consider using the no-threads shim mode in rayon 1.7.0
  instead of `maybe-rayon`.

The repo does not use a `good-first-issue` label, but
`A-documentation` and `A-book` issues are reasonable first PRs.

## Canonical external references

- Halo paper: <https://eprint.iacr.org/2019/1021>.
- PLONK paper: <https://eprint.iacr.org/2019/953>.
- Plookup: <https://eprint.iacr.org/2020/315>.
- Pasta curves: <https://github.com/zcash/pasta_curves>.
- Poseidon: <https://eprint.iacr.org/2019/458>.
- Sinsemilla: ZIP 224 and the halo2 book chapter.
- IPA without trusted setup: Bowe, Grigg, Hopwood (2019);
  Bunz et al., "Recursive Proof Composition without a Trusted
  Setup".
- The Halo 2 Book (mdbook): <https://zcash.github.io/halo2/>.

## Chapter graph (derived)

The reader must understand the math and types in this order:

1. Workspace map (00).
2. Build and contribute loop (01).
3. Fields, FFT, MSM (02), since every later chapter assumes them.
4. Polynomial domains (03), prerequisite for proving / commitment.
5. PLONKish arithmetization (04), prerequisite for circuit
   synthesis.
6. Circuit synthesis (05), prerequisite for gadgets.
7. Lookup argument (06).
8. Permutation argument (07).
9. IPA polynomial commitment (08).
10. Multiopen (09).
11. Prover / verifier flow (10), stitches 06-09 together.
12. Fiat-Shamir transcripts (11), referenced throughout.
13. Dev tooling (12).
14. Gadgets organization + ECC chip (13).
15. Hash gadgets: Poseidon, Sinsemilla, SHA-256 (14).
16. Study plan and contribution scenarios (15).

Index page is `index.md` (sidebar_position 0). Chapters are
numbered `NN-slug.md` from `01` upward.
