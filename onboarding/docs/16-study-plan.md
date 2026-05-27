---
sidebar_position: 16
title: Study Plan and Contribution Scenarios
description: A two-week schedule that converges on a real PR you can plausibly open.
---

# Study Plan and Contribution Scenarios

## 1. Why this chapter exists

A reader who has worked through chapters 01 to 15 has the
vocabulary to read halo2's source. They do not yet have the
muscle memory to write a PR. This chapter is a two-week schedule
that turns the reading into a deliverable: by the end of week
two, you have a small, well-scoped change ready to send upstream.
Every day's plan ends with an exercise that touches code and a
short reflection.

## 2. Definitions

**Definition (Reading day).** A day spent reading a chapter and
following the exercises end-to-end. Two to four hours.

**Definition (Hands-on day).** A day spent modifying or
extending the code, building, testing, and writing a one-page
note on what you changed and why.

**Definition (Contribution day).** A day spent writing a single,
mergeable patch: a fix, a test, a documentation pass, a small
refactor. The patch should pass the full local CI command from
chapter 02.

## 3. The plan

### Week 1: read and run

| Day | Activity | Chapter |
| --- | --- | --- |
| Mon | Read; run `cargo test --release -p halo2_proofs`. Note timings. | 01, 02 |
| Tue | Read; run benchmarks; rederive `Params(8)` deterministically. | 03, 04 |
| Wed | Read; modify `simple-example` with an extra gate; re-run. | 05 |
| Thu | Read; switch `simple-example` to `V1` planner; observe layout. | 06 |
| Fri | Read; trigger a `Permutation` failure in `MockProver`. | 07, 08 |

End-of-week 1 deliverable: a private notebook with one sentence
per chapter on the most surprising thing you learned and the
location in the code that surprised you. Keep it; it is the
artefact that turns reading into knowledge.

### Week 2: contribute

| Day | Activity | Chapter |
| --- | --- | --- |
| Mon | Read; trace one IPA round in the prover; explain to yourself why $L$ and $R$ are committed before the challenge. | 09 |
| Tue | Read; read the 0.3.1 changelog; rederive (on paper) why the multiopen fix was necessary. | 10 |
| Wed | Read; instrument the prover with `tracing::info!`; observe the order of transcript writes. | 11, 12 |
| Thu | Read; run `circuit-layout` for a non-trivial gadget; record the layout density. | 13 |
| Fri | Pick a contribution scenario from below; open a PR. | 14, 15 |

End-of-week 2 deliverable: one open PR. It can be tiny. Tiny PRs
are better; they teach the review loop.

## 4. Contribution scenarios

Each scenario is sized for an experienced Rust developer with one
week of halo2 reading. Each lists the chapters that prepare you.

### 4.1 Documentation improvement

- Source: any open `A-book` issue from
  [zcash/halo2/issues](https://github.com/zcash/halo2/issues?q=is%3Aissue+is%3Aopen+label%3AA-book).
- Touches: `book/src/...`.
- Prerequisites: chapter 02 (so you can run `mdbook test`).
- Reviewer focus: clarity, accuracy, mdbook formatting.
- Example issues: #800 (Sinsemilla page rendering), #810
  (compatibility documentation).

### 4.2 New `MockProver` failure-message test

- Source: pick a chip in `halo2_gadgets/src/`. Identify a
  constraint that is hard to debug from the current
  `VerifyFailure` output.
- Touches: a new `#[test]` in the chip's test module that
  demonstrates the failure and asserts on the exact failure
  message.
- Prerequisites: chapters 06, 13.
- Reviewer focus: the test must actually fail without your
  improvement and pass with it.

### 4.3 Constant-time review of `lookup_range_check`

- Source: read
  [`halo2_gadgets/src/utilities/lookup_range_check.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/utilities/lookup_range_check.rs)
  and the issue
  [#820](https://github.com/zcash/halo2/issues/820).
- Touches: clarifying comments, or a tightened constraint with
  an extra test.
- Prerequisites: chapters 05, 07, 14.
- Reviewer focus: the change should preserve the existing
  invariant. The chip is used by ECC and Sinsemilla; do not
  break either.

### 4.4 Benchmark addition

- Source: the FFT or MSM bench in
  [`halo2_proofs/benches/`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/benches).
- Touches: a new Criterion bench group that exercises an edge
  case (e.g. very small or very large inputs).
- Prerequisites: chapter 03.
- Reviewer focus: numeric stability of the measurement; no
  spurious noise from setup.

### 4.5 A reproduction harness for a `dev-graph` regression

- Source: run `circuit-layout` against a non-trivial circuit
  (the `two-chip` example) and compare against the saved
  reference image
  [`halo2_proofs/examples/circuit-layout.png`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/examples/circuit-layout.png).
- Touches: a small script in `book/` or a doctest.
- Prerequisites: chapter 13.

## 5. The PR checklist (from chapter 02, repeated here)

Before pushing, run locally:

1. `cargo fmt --all` and verify with `cargo fmt --all -- --check`.
2. `cargo clippy --no-default-features --features 'batch dev-graph
   gadget-traces test-dependencies' --all-targets -- -D warnings`.
3. The CI test command from chapter 02.
4. `cargo doc --workspace --all-features --document-private-items`.
5. Update the relevant `CHANGELOG.md` under `## [Unreleased]`.
6. Open the PR with a description that links the issue (if any),
   summarizes the change, and notes any compatibility
   considerations (verifying-key impact, MSRV, etc.).

## 6. Failure modes

- **Skipping the reading and going straight to the
  contribution.** The chip code is dense enough that small
  changes can introduce soundness bugs invisible to local tests.
  Read first; touch second.
- **Choosing a contribution that is too large.** Two days for a
  first PR is realistic; two weeks is not. If the scope
  threatens to grow, split it.
- **Not running the full CI command locally.** The maintainers
  run it on every PR; if your local matches, you waste no
  cycles on red CI.
- **Skipping the changelog entry.** It is required and CI gates
  on it.

## 7. Exercises

1. Pick one chapter from 01-15 and re-read it without the source
   open. Write the chapter's main definitions from memory; check
   against the chapter and against the code.
2. Pick one of the contribution scenarios above. Identify, in
   one paragraph, exactly which file you would touch and which
   tests you would need to add or update.
3. Build the docs locally:

   ```bash
   cd onboarding
   make install
   make build
   make preview
   ```

   Confirm the site renders the chapters in order.
4. Find one error in this onboarding course (a stale line
   number, a typo, a wrong file path) and open a PR against the
   [`onboarding` branch of the fork](https://github.com/dannywillems/halo2/tree/onboarding).
   This counts as your first contribution.

## 8. Spec pointers

None new. This chapter integrates the references from chapters
01 to 15.

## 9. Further reading

- [Electric Coin Company's halo2 release notes](https://github.com/zcash/halo2/releases)
  for the changes the maintainers care about most.
- [zkSecurity audit reports](https://www.zksecurity.xyz/blog) for
  examples of the kind of issues a careful reader can find.

