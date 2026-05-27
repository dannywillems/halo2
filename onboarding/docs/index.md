---
sidebar_position: 0
title: Welcome
description: Auto-generated, code-anchored onboarding course for the zcash/halo2 PLONKish proving system.
---

# halo2 Onboarding

This course is a contribution-oriented walk through
[zcash/halo2](https://github.com/zcash/halo2), pinned at commit
[`32a8758`](https://github.com/zcash/halo2/commit/32a87582dfb0ad9364ef3ffe71751ceab2a502ea).
The goal is to take a reader from "I have read the README" to "I can
open a small PR" in two focused weeks. Every chapter is anchored to
specific files and line ranges in the upstream repository.

:::warning Auto-generated; verify against source

This site was generated automatically using
[Claude Code](https://claude.com/claude-code) and may contain
inaccuracies, paraphrased proofs, or stale references. **The code in
the repository is the law.** The authoritative references for this
course are:

- The halo2 source tree, pinned at commit
  [`32a8758`](https://github.com/zcash/halo2/commit/32a87582dfb0ad9364ef3ffe71751ceab2a502ea).
- The official
  [Halo 2 Book](https://zcash.github.io/halo2/), maintained by the
  Electric Coin Company.
- The Halo paper
  ([eprint 2019/1021](https://eprint.iacr.org/2019/1021))
  and the PLONK paper
  ([eprint 2019/953](https://eprint.iacr.org/2019/953)).
- The crate documentation on
  [docs.rs/halo2_proofs](https://docs.rs/halo2_proofs) and
  [docs.rs/halo2_gadgets](https://docs.rs/halo2_gadgets).

If you spot a mistake, please open an issue or PR against the
[`onboarding` branch of the fork](https://github.com/dannywillems/halo2/tree/onboarding).

:::

## How to read this course

The chapters are designed to be read in order, but each one is
self-contained enough to be referenced in isolation. The dependency
graph the chapter ordering encodes:

1. **Workspace map and contribution loop** (chapters 01 and 02): you
   need this to navigate the code and to push a PR.
2. **Field arithmetic, FFT, MSM, polynomial domains** (03 and 04):
   the algebra layer every later chapter assumes.
3. **PLONKish arithmetization and circuit synthesis** (05 and 06):
   what circuit authors actually write, and how it is laid out.
4. **Lookup, permutation, IPA, multiopen** (07 to 10): the proving
   system's argument tools.
5. **Prover / verifier flow and transcripts** (11 and 12): stitch
   the arguments together with Fiat-Shamir.
6. **Dev tooling** (13): `MockProver`, `CircuitCost`,
   `CircuitGates`, dev-graph.
7. **halo2_gadgets** (14 and 15): ECC, Poseidon, Sinsemilla,
   SHA-256.
8. **Study plan and contribution scenarios** (16): two weeks of
   exercises that converge on a real PR.

If you only have one hour, read chapter 02 (the build loop) and the
PLONKish arithmetization in chapter 05.

## Notation

Throughout this course we use the following conventions.

- $\mathbb{F}_p$: the Pallas scalar field. The default field used
  by every chapter is `pasta_curves::Fp`.
- $\mathbb{G}$: the Pallas curve, written additively. Group elements
  are points $P, Q \in \mathbb{G}$; the group operation is $P + Q$
  and the identity is $\mathcal{O}$.
- $[k]P$: scalar multiplication of $P \in \mathbb{G}$ by
  $k \in \mathbb{F}_p$.
- $n = 2^k$: the size of the evaluation domain. The PLONKish circuit
  has $n$ rows.
- $\omega \in \mathbb{F}_p$: a primitive $n$-th root of unity.
- $\zeta \in \mathbb{F}_p$: an extension factor (third root of unity)
  used to build the extended domain.
- $L_i(X)$: the $i$-th Lagrange basis polynomial,
  $L_i(\omega^j) = \delta_{ij}$.
- $\mathsf{Com}(p)$: a Pedersen commitment to a polynomial $p$,
  see chapter 09.
- $\mathsf{Hash}(\cdot) = \mathsf{Blake2b}_{512}(\cdot)$: the
  transcript hash (see chapter 12).
- $a \mathbin{\|} b$: byte-string concatenation.
- $a \stackrel{\$}{\leftarrow} S$: $a$ sampled uniformly from $S$.

## Prerequisites

You should be comfortable with:

- Rust 1.60 or later, including traits, generics, and lifetimes.
- Basic finite-field algebra: prime fields, multiplicative
  subgroups, roots of unity, the Fast Fourier Transform.
- One mental model of a polynomial commitment scheme. KZG is fine;
  the IPA used here is built up from scratch in chapter 09.
- One mental model of a SNARK. PLONK is a plus; chapter 05 reviews
  the PLONKish arithmetization.

If any of these are unfamiliar, the
[Halo 2 Book "Background Material"](https://zcash.github.io/halo2/background.html)
section is a good warm-up.

## License

The course content is licensed under MIT or Apache 2.0, matching the
upstream halo2 license. The upstream halo2 code is Copyright (c)
Electric Coin Company and contributors.
