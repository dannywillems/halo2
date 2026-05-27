---
sidebar_position: 2
title: Build, Test, and Contribute
description: The toolchain, the test loop, and what a halo2 PR looks like in practice.
---

# Build, Test, and Contribute

## 1. Why this chapter exists

You can read every other chapter without ever running a command,
and you will get nothing useful out of it. Code understanding
without an executable loop is theatre. This chapter exists so that
by the end of it, you can:

- compile the workspace,
- run a focused test against a single function,
- run the whole CI matrix locally,
- and reproduce the four checks that CI gates a PR on (test, fmt,
  clippy, docs).

It also identifies the files that contributors actually touch, so
you know where to look for examples of what good change sets look
like.

## 2. Definitions

**Definition (MSRV).** Minimum Supported Rust Version. Set in
[`rust-toolchain.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/rust-toolchain.toml)
to `1.60.0`. The MSRV is binding: a PR that requires a newer
compiler is rejected.

**Definition (Hot file).** A source file modified more than twice in
the last twelve months. These are the files where most contributions
land; the list is in `discovery.md`.

**Definition (Feature flag set).** The exact set of `--features`
flags CI uses. Defined in
[`.github/actions/prepare/action.yml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/.github/actions/prepare/action.yml).

## 3. The code

### 3.1 The toolchain

```toml file=../../rust-toolchain.toml title="rust-toolchain.toml"
```
Install rustup, then `cd` into the repo. The toolchain file will
pin your compiler to `1.60.0` automatically. Two facts to keep in
mind:

- `clippy` and `rustfmt` are listed as required components; rustup
  will install them on first use.
- Many crates in the wider ecosystem have moved past MSRV 1.60. The
  workspace pins indirect-dev dependencies (`dashmap`, `image`,
  `tempfile`) to versions that still build on 1.60, as documented in
  [`halo2_proofs/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml).
  Do not bump them without coordinating an MSRV update.

### 3.2 The feature flags CI uses

The composite action that all CI jobs share is:

```yaml file=../../.github/actions/prepare/action.yml title=".github/actions/prepare/action.yml"
```
The flag string is:

```
--no-default-features --features 'batch dev-graph gadget-traces [beta] [nightly] [test-dependencies]'
```

To reproduce a CI test run locally:

```bash
cargo test --release --workspace \
    --no-default-features \
    --features 'batch dev-graph gadget-traces test-dependencies'
```

For the beta and nightly variants, append `--features beta` or
`--features nightly`.

### 3.3 Test layout

- Unit tests are inline in the files they test; search for
  `#[cfg(test)] mod tests` at the bottom of any `.rs` file.
- Integration tests live in `halo2_proofs/tests/`. The main one is
  [`plonk_api.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/tests/plonk_api.rs)
  with a checked-in proof fixture
  [`plonk_api_proof.bin`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/tests/plonk_api_proof.bin).
  Any change that alters the byte layout of a proof must regenerate
  this fixture or fail CI.
- Examples (`halo2_proofs/examples/`) are also compiled by CI under
  the `bitrot` job.
- Benchmarks live in `halo2_proofs/benches/` with `harness = false`
  declared in `Cargo.toml`; run with `cargo bench`.

### 3.4 Local commands you will use daily

```bash
# Build the workspace in debug mode.
cargo build --workspace

# Run all tests with the CI feature set.
cargo test --release --workspace \
    --no-default-features \
    --features 'batch dev-graph gadget-traces test-dependencies'

# Run one test by name (substring match).
cargo test --release -p halo2_proofs plonk_api -- --nocapture

# Format check (CI-equivalent).
cargo fmt --all -- --check

# Format in place.
cargo fmt --all

# Clippy (MSRV) with all targets and warnings denied.
cargo clippy --no-default-features \
    --features 'batch dev-graph gadget-traces test-dependencies' \
    --all-targets -- -D warnings

# Run the simple-example.
cargo run --release --example simple-example -p halo2_proofs

# Build the rendered API docs.
cargo doc --workspace --all-features --document-private-items --open
```

### 3.5 The CI graph

The PR checks are split across several workflows:

```yaml file=../../.github/workflows/ci.yml title=".github/workflows/ci.yml"
```
The notable jobs:

- `test`: matrix over `{stable, beta, nightly}` x
  `{ubuntu, windows, macOS}`, running
  `cargo test --release --workspace`.
- `test-32-bit`: i686 cross-compile and test. Catches usize-as-32-bit
  bugs.
- `build`: cross-compiles to `wasm32-unknown-unknown` and
  `wasm32-wasi`. WASM-incompatible imports break here.
- `bitrot`: builds benchmarks and examples with `--all-features`
  using the stable toolchain.
- `book`: runs `mdbook test` against the `book/` directory; the
  book includes runnable Rust examples that must compile.
- `codecov`: runs `cargo tarpaulin` with nightly features and
  uploads coverage.
- `doc-links`: runs `cargo doc --all --all-features
  --document-private-items` with `#![deny(rustdoc::broken_intra_doc_links)]`.
- `fmt`: `cargo fmt --all -- --check`.
- `lints-stable.yml` and `lints-beta.yml`: clippy at MSRV / beta with
  `-D warnings`.

### 3.6 Hot files (where contributions land)

The most-changed files in the last twelve months. Each is a likely
landing point for newcomer PRs:

- [`halo2_proofs/src/poly/multiopen.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/multiopen.rs):
  the multiopen wrapper; modified by the 0.3.1 soundness fix.
- [`halo2_gadgets/src/utilities/lookup_range_check.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/utilities/lookup_range_check.rs):
  a heavily reused range-check helper.
- [`halo2_proofs/src/poly/multiopen/prover.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/poly/multiopen/prover.rs):
  the multiopen prover.
- [`halo2_gadgets/src/sinsemilla/merkle.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs):
  the Sinsemilla-Merkle path verifier.
- [`halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/chip/hash_to_point.rs):
  the inner loop of the Sinsemilla chip.
- [`halo2_proofs/src/arithmetic.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/arithmetic.rs):
  `best_multiexp` and FFT helpers.

### 3.7 Good first issues

The repo does not use a `good-first-issue` label, but two
categories of open issues are within reach of a newcomer:

- `A-book` issues
  ([example: #800](https://github.com/zcash/halo2/issues/800),
  [#810](https://github.com/zcash/halo2/issues/810)): they only
  require editing `book/src/...`.
- Sanity-check or documentation improvements
  (e.g. [#778](https://github.com/zcash/halo2/issues/778),
  [#794](https://github.com/zcash/halo2/issues/794)): small,
  well-scoped edits.

A complete list is at
[zcash/halo2/issues](https://github.com/zcash/halo2/issues).

### 3.8 The PR checklist

Before pushing a PR, run locally:

1. `cargo fmt --all` and verify with `cargo fmt --all -- --check`.
2. `cargo clippy --no-default-features --features 'batch dev-graph
   gadget-traces test-dependencies' --all-targets -- -D warnings`.
3. The CI test command above.
4. `cargo doc --workspace --all-features --document-private-items`
   to verify no broken intra-doc links.
5. If you touched any file that affects proof bytes, regenerate
   `halo2_proofs/tests/plonk_api_proof.bin`.
6. Update the relevant `CHANGELOG.md` in either `halo2_proofs/` or
   `halo2_gadgets/`. The entry goes under `## [Unreleased]`.

The repo does not require signed commits, but it does require
Apache-2.0 / MIT dual-licensing of contributions (the standard
sentence is in
[`README.md`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/README.md)).

## 4. Failure modes

- **Forgetting `--release`.** Running `cargo test` without
  `--release` takes 10 to 100x longer because polynomial FFTs
  dominate runtime. The CI matrix always uses `--release`; you
  should too.
- **Forgetting the feature flags.** Running `cargo test` with
  default features omits `dev-graph` (and so omits a number of
  tests). Always use the CI flag string above.
- **Bumping an indirect-dev dependency past MSRV.** The pins in
  `halo2_proofs/Cargo.toml` exist precisely to keep MSRV at 1.60.
  If a tool nags you to update one, check its new MSRV first.
- **Editing `plonk_api_proof.bin` by hand.** This file is binary
  and is regenerated by the test suite when you set the right env
  var; never edit it manually.

## 5. Spec pointers

- The
  [GitHub Actions documentation on composite actions](https://docs.github.com/en/actions/creating-actions/creating-a-composite-action)
  explains the pattern used in
  [`.github/actions/prepare/`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/.github/actions/prepare/action.yml).
- The Rust toolchain file format is documented in the
  [rustup book](https://rust-lang.github.io/rustup/overrides.html#the-toolchain-file).

## 6. Exercises

1. Run `cargo test --release -p halo2_proofs plonk_api` from a
   clean clone. Time how long it takes. Report which test names
   were executed.
2. Find the line in
   [`halo2_proofs/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml)
   that pins `dashmap` to `< 5.5.0`. Note the inline comment that
   explains why; this is the canonical example of an MSRV-driven
   indirect-dep pin.
3. Pick one open `A-book` issue from
   [zcash/halo2/issues](https://github.com/zcash/halo2/issues?q=is%3Aissue+is%3Aopen+label%3AA-book).
   In two sentences each, identify (a) which book file would need
   to change, (b) which chapter of this course covers the same
   material.
4. Open
   [`halo2_proofs/CHANGELOG.md`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/CHANGELOG.md)
   and find the `0.3.1` entry. Note what the soundness bug was;
   you will see this again in chapter 10 (multiopen).

### Answers in the code

- Exercise 1: the matching test names are inside
  [`halo2_proofs/tests/plonk_api.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/tests/plonk_api.rs).
- Exercise 2: the `dashmap = ">=5, <5.5.0"` line is in the
  `[dev-dependencies]` section of
  [`halo2_proofs/Cargo.toml`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/Cargo.toml)
  with comment `# dashmap 5.5.0 has MSRV 1.64`.
- Exercise 4: the `0.3.1` security section of
  [`halo2_proofs/CHANGELOG.md`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/CHANGELOG.md).
