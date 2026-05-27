---
sidebar_position: 0.5
title: 'Background: From Zerocoin to Orchard'
description: The shielded-pool primitive vocabulary (note, commitment, anchor, nullifier) and the lineage Zerocoin -> Zerocash -> Sprout, Sapling, Orchard. Reading this first makes the gadget chapters make sense.
---

# Background: From Zerocoin to Orchard

## 1. Why this chapter exists

`halo2_proofs` is a generic proving system; `halo2_gadgets` is a
library of chips. Neither file mentions Zcash. But every design
choice in `halo2_gadgets`, from the choice of Pallas + Vesta to
the Sinsemilla hash to the lookup-range-check widths, was made to
support the Orchard shielded pool. Without knowing what an
*Orchard note* or a *nullifier* is, the reader has no way to
judge why the ECC chip exposes
[`mul_fixed_short`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/ecc/chip/mul_fixed/short.rs),
why Sinsemilla matters more than SHA-256 here, or why the
specific constant `MERKLE_CRH_PERSONALIZATION =
"z.cash:Orchard-MerkleCRH"` lives in
[`sinsemilla/merkle.rs#L16`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs#L16).

This chapter is the bridge. It walks the lineage from the
original Zerocoin protocol (2013), through Zerocash (2014), and
into the three Zcash shielded pools (Sprout, Sapling, Orchard).
It pins down the vocabulary used in chapters 14 (ECC) and 15
(hash gadgets), so when those chapters say "the chip computes a
nullifier", you already know what a nullifier is and why it
needs to be a curve point.

If you only ever read one section of this chapter, read
[Section 4](#4-the-shielded-pool-vocabulary): the primitive
vocabulary.

## 2. Definitions

We use Zerocash / Zcash terminology throughout. Capital-script
letters are curve points, lowercase italics are scalars.

**Definition (Anonymous payment scheme).** A protocol that lets
a payer transfer value to a payee such that an observer of the
ledger learns no more than is strictly necessary to verify
consensus rules (typically: value conservation, no
double-spending). The strongest schemes (Zerocash and successors)
hide the amount, the sender, and the receiver of every shielded
transaction.

**Definition (Zerocoin).** The first proposal for SNARK-free
anonymous payments on top of a Bitcoin-style ledger, due to Miers,
Garman, Green, and Rubin
([IEEE S&P 2013](https://ieeexplore.ieee.org/document/6547123)
[[PDF](https://zerocoin.org/media/pdf/ZerocoinOakland.pdf)]). Uses
an RSA accumulator and double-discrete-log proofs. Coins are
fixed-denomination and *not* hidden in amount; only the linkage
between mint and spend is hidden. Zcash does not implement
Zerocoin; Zcash implements Zerocash, the SNARK-based successor.

**Definition (Zerocash).** The zk-SNARK-based successor of
Zerocoin, due to Ben-Sasson, Chiesa, Garman, Green, Miers,
Tromer, and Virza
([IEEE S&P 2014](https://www.ieee-security.org/TC/SP2014/papers/Zerocash_c_DecentralizedAnonymousPaymentsfromBitcoin.pdf))
[[PDF](http://zerocash-project.org/media/pdf/zerocash-extended-20140518.pdf)].
Hides amounts and addresses by representing payments as commitments
("notes") that are added to a Merkle tree; spends prove "I know the
opening of a note in the tree" using a SNARK, and publish a
deterministic *nullifier* per spent note to prevent double-spends.
This is the construction Zcash inherited.

**Definition (Shielded pool).** A subset of UTXOs whose value
lives behind the Zerocash construction. A wallet "shields" funds
by spending a transparent UTXO and creating shielded notes; it
"deshields" by spending shielded notes and creating transparent
outputs. The pool is the set of notes whose commitments have
been published and whose nullifiers have not.

**Definition (Action).** Orchard's atomic transaction primitive.
An action is the symmetric combination of one note spend and one
note output, with shared randomization. Sapling separated spends
from outputs (`SpendDescription` and `OutputDescription`); Orchard
fused them into a single `ActionDescription`, simplifying the
circuit and the transaction format. See
[ZIP 224, "Orchard Shielded Protocol"](https://zips.z.cash/zip-0224).

## 3. From Zerocoin to Orchard: the lineage

### 3.1 Zerocoin (2013)

- **Crypto:** RSA accumulator, Schnorr-style double-discrete-log
  proofs.
- **What's hidden:** the link between a mint and a spend (one-of-N
  anonymity over the entire mint set).
- **What's not hidden:** the amount (each coin is a fixed
  denomination), and the fact that a shielded operation took place.
- **Cost:** proof size around 25 KB per spend; verification
  fractions of a second.

Reference: Miers et al., 2013.

### 3.2 Zerocash (2014)

- **Crypto:** Groth16-style preprocessing zk-SNARKs (later Pinocchio
  / BCTV14 / Groth16), an R1CS arithmetization, and a trusted setup.
- **What's hidden:** amount, sender, receiver, and the link
  between input and output notes. Only the existence of a shielded
  transaction is public.
- **What's not hidden:** transaction fee (in early Zcash), and
  any value transferred between shielded and transparent pools.
- **Cost:** proof size around 300 bytes; verification under 10 ms.

Reference: Ben-Sasson et al., 2014.

### 3.3 Sprout (Zcash launch, Oct 2016)

The first deployed shielded pool. A direct implementation of
Zerocash with a few Zcash-specific tweaks (JoinSplit transactions,
note structure, address format).

| Aspect              | Value                                          |
| ------------------- | ---------------------------------------------- |
| Curve               | alt_bn128 (Barreto-Naehrig, 254-bit)           |
| Proof system        | BCTV14, later Groth16 via Sapling backport     |
| Hash for commitments | SHA-256 truncated to 256 bits                  |
| Tree commitment hash | SHA-256-based PRF                              |
| Trusted setup       | "Powers of Tau" / "Sprout MPC", 6 participants |
| Transaction format  | JoinSplit (2-in / 2-out per JoinSplit)         |
| Spec section        | Zcash Protocol Spec section 4                  |

Sprout is in deprecation: new value cannot enter the pool, only
exit. The Sprout circuit is in
[zcash/librustzcash](https://github.com/zcash/librustzcash); it is
*not* a halo2 circuit.

### 3.4 Sapling (Network Upgrade 1, Oct 2018)

Redesigned for circuit efficiency and for separating
spend-authorisation from note-decryption keys.

| Aspect              | Value                                                                 |
| ------------------- | --------------------------------------------------------------------- |
| Curve               | BLS12-381 (proving) + JubJub (in-circuit)                             |
| Proof system        | Groth16                                                               |
| Hash for commitments | Pedersen hash over JubJub                                             |
| Tree commitment hash | Pedersen hash                                                         |
| Trusted setup       | "Sapling MPC", ~90 participants                                       |
| Transaction format  | `SpendDescription` (one per input) + `OutputDescription` (per output) |
| Spec section        | Zcash Protocol Spec section 4 (Sapling parts)                         |
| Notable additions   | Diversified addresses; full viewing keys; spend-authority signatures  |

Sapling is the most-used shielded pool at the time of writing.
The Sapling circuit lives in
[zcash/sapling-crypto](https://github.com/zcash/sapling-crypto) and
its bellman-based proving code in the bellman library. The
Sapling circuit is **not** a halo2 circuit.

### 3.5 Orchard (Network Upgrade 5, May 2022)

Redesigned to remove the trusted setup and to share infrastructure
with the planned recursive proof system. This is where halo2
appears.

| Aspect              | Value                                                  |
| ------------------- | ------------------------------------------------------ |
| Curves              | Pallas + Vesta (the Pasta cycle)                       |
| Proof system        | halo2 PLONK + IPA (this repository)                    |
| Note commitment hash | Sinsemilla                                             |
| Tree commitment hash | MerkleCRH^Orchard (Sinsemilla)                         |
| Nullifier PRF       | Poseidon over Pallas                                   |
| Trusted setup       | **none** (transparent setup via Halo's IPA)            |
| Transaction format  | `ActionDescription` (one per spend+output bundle)      |
| Spec section        | Zcash Protocol Spec section 4 (Orchard parts), [ZIP 224](https://zips.z.cash/zip-0224) |

The Orchard circuit lives in
[zcash/orchard](https://github.com/zcash/orchard); the chips it
uses live in `halo2_gadgets` (this repository). Chapters 14 and
15 of this course walk those chips one by one.

### 3.6 At a glance

| Property         | Zerocoin     | Zerocash       | Sprout        | Sapling        | Orchard            |
| ---------------- | ------------ | -------------- | ------------- | -------------- | ------------------ |
| SNARK            | no           | yes            | yes (BCTV14)  | yes (Groth16)  | yes (halo2 + IPA)  |
| Trusted setup    | no           | yes (per-pair) | yes (MPC)     | yes (MPC)      | **no**             |
| Amounts hidden   | no           | yes            | yes           | yes            | yes                |
| Addresses hidden | no           | yes            | yes           | yes            | yes                |
| Curve            | RSA          | various        | alt_bn128     | BLS12-381      | Pallas / Vesta     |
| Note hash        | n/a          | SHA-256        | SHA-256       | Pedersen       | Sinsemilla         |
| Year             | 2013         | 2014           | 2016          | 2018           | 2022               |

## 4. The shielded-pool vocabulary

Every term below appears verbatim in the Orchard chip code. The
formulas use Orchard notation; the Sapling analogues are similar
modulo the choice of hash and curve.

### 4.1 Note

A record describing a single shielded UTXO. In Orchard, a note is
the tuple
$$\mathsf{note} = (\mathsf{d}, \mathsf{pk_d}, v, \rho, \psi, \mathsf{rcm})$$
with:

- $\mathsf{d}$: an 11-byte *diversifier* (address randomization).
- $\mathsf{pk_d}$: the recipient's diversified transmission key, a
  Pallas point.
- $v$: the note value, a 64-bit unsigned integer.
- $\rho$: a 255-bit field element binding the note to a specific
  prior nullifier (this is what links action chains).
- $\psi$: a 255-bit "additional randomness" derived from $\rho$
  and the random seed.
- $\mathsf{rcm}$: the commitment trapdoor, a 255-bit field
  element.

Sapling's note has the same shape but without an explicit $\rho$
or $\psi$ field; those role split differently.

### 4.2 Note commitment

A binding, hiding commitment to a note, published on chain:
$$
\mathsf{cm} \;=\; \mathsf{NoteCommit}^{\mathsf{Orchard}}_{\mathsf{rcm}}\bigl(\mathsf{repr}_{\mathbb{P}}(\mathsf{g_d}),\; \mathsf{repr}_{\mathbb{P}}(\mathsf{pk_d}),\; v,\; \rho,\; \psi\bigr)
$$
where $\mathsf{g_d}$ is the diversifier base point ($\mathsf{g_d}
= \mathrm{DiversifyHash}(\mathsf{d})$) and
$\mathsf{NoteCommit}^{\mathsf{Orchard}}$ is a SinsemillaCommit
([Zcash Protocol Spec section 5.4.8.4](https://zips.z.cash/protocol/protocol.pdf#concretesinsemillacommit)).
Implemented in the Orchard circuit using
[`halo2_gadgets::sinsemilla`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla.rs).

### 4.3 Note commitment tree and anchor

Note commitments are appended to an incremental Merkle tree of
fixed depth `MERKLE_DEPTH_ORCHARD = 32`:

```rust file=../../halo2_gadgets/src/sinsemilla/merkle.rs#L14-L18 title="halo2_gadgets/src/sinsemilla/merkle.rs"
```

The hash at each layer is `MerkleCRH^Orchard`, a 510-bit-input
Sinsemilla hash; the personalization string above is the
canonical domain tag the Orchard spec mandates. The **anchor** is
the root of this tree at some past block height; an Orchard
spend proves membership of one of the leaves under the anchor
without revealing which.

### 4.4 Nullifier

The unique, unlinkable identifier of a spent note:
$$
\mathsf{nf} \;=\; \mathrm{Extract}_{\mathbb{P}}\bigl([\mathrm{PRF}^{\mathsf{nfOrchard}}_{\mathsf{nk}}(\rho) + \psi \bmod p]\, K^{\mathsf{Orchard}} \;+\; \mathsf{cm}\bigr)
$$
where:

- $\mathsf{nk}$ is the nullifier-deriving key (private to the
  spender).
- $\mathrm{PRF}^{\mathsf{nfOrchard}}$ is implemented with Poseidon
  ([Zcash Protocol Spec section 5.4.2](https://zips.z.cash/protocol/protocol.pdf#concreteprfs)).
- $K^{\mathsf{Orchard}}$ is a fixed base point.
- $\mathrm{Extract}_{\mathbb{P}}$ extracts the $x$-coordinate.

Publishing $\mathsf{nf}$ on chain marks the note as spent.
Without $\mathsf{nk}$, an observer cannot compute $\mathsf{nf}$
from $\mathsf{cm}$; with $\mathsf{nk}$, the wallet can. This is
what makes shielded chains both unlinkable and double-spend-safe.

### 4.5 Value commitment

A Pedersen commitment to the note value, in Pallas:
$$
\mathsf{cv} \;=\; [v]\,\mathcal{V}^{\mathsf{Orchard}} \;+\; [\mathsf{rcv}]\,\mathcal{R}^{\mathsf{Orchard}}.
$$
Action descriptions publish $\mathsf{cv}$ for inputs and outputs.
Consensus checks that the sum of input value commitments minus
the sum of output commitments equals
$[v_{\mathsf{balance}}] \mathcal{V} + [0] \mathcal{R}$ for the
declared net value transferred between the shielded and
transparent pools, which is enough to verify amount conservation
without learning individual values.

### 4.6 Diversifier and ivk

The full viewing key derives a per-address *diversifier* $\mathsf{d}$
that randomizes the address without changing the underlying
spending key. The incoming viewing key is
$$
\mathsf{ivk} \;=\; \mathrm{Commit}^{\mathsf{ivk}}_{\mathsf{rivk}}(\mathsf{ak},\; \mathsf{nk})
$$
where $\mathsf{ak}$ is the spend-authorization public key and
$\mathsf{Commit}^{\mathsf{ivk}}$ is a `SinsemillaCommit`.

### 4.7 Action

The Orchard transaction primitive. An `ActionDescription` bundles:

- The input note's nullifier $\mathsf{nf_{old}}$.
- The output note's commitment $\mathsf{cm_{new}}$.
- An input value commitment $\mathsf{cv_{old}}$ and an output
  $\mathsf{cv_{new}}$.
- A randomized spend-auth verification key $\mathsf{rk}$.
- An encrypted ciphertext containing the new note (so the
  recipient can decrypt with their ivk).
- A halo2 proof that all of the above are consistent with some
  $(\mathsf{anchor}, v_{\mathsf{balance}}, ...)$ public input.

The Orchard *circuit* is the predicate that proof certifies. It is
defined in
[zcash/orchard/src/circuit.rs](https://github.com/zcash/orchard/blob/main/src/circuit.rs)
and is the largest known consumer of `halo2_gadgets`.

## 5. How the halo2_gadgets chips map to Orchard primitives

| Orchard primitive                | Halo2 chip / gadget                                              |
| -------------------------------- | ---------------------------------------------------------------- |
| `NoteCommit^Orchard`             | `halo2_gadgets::sinsemilla::CommitDomain` (Sinsemilla commit)    |
| `MerkleCRH^Orchard`              | `halo2_gadgets::sinsemilla::merkle::MerklePath`                  |
| `PRF^nfOrchard`                  | `halo2_gadgets::poseidon::Hash` (`Pow5Chip`)                     |
| `Commit^ivk`                     | `halo2_gadgets::sinsemilla::CommitDomain`                        |
| Value commitment `[v]V + [r]R`   | `halo2_gadgets::ecc::EccChip` (variable + fixed-base scalar mul) |
| `DiversifyHash`                  | (Group-hash-to-curve, outside the in-circuit chip set)           |
| `Extract_P`                      | `halo2_gadgets::ecc::X` (x-coordinate extraction)                |
| Spend-auth randomization         | `halo2_gadgets::ecc::EccChip::mul`                               |
| Range checks on $v$, $\rho$, etc. | `halo2_gadgets::utilities::lookup_range_check`                  |
| `Mixing^{Pedersen}` (Sapling)    | (Sapling only; not in `halo2_gadgets`)                           |

The bottom four rows of this table are why the ECC chip
(chapter 14) has the exact API it has. The top three rows are why
the hash gadgets chapter (chapter 15) emphasizes Sinsemilla and
Poseidon, rather than the more familiar SHA-256.

## 6. Failure modes

- **Cross-pool confusion.** Sprout, Sapling, and Orchard each
  have their own note structure, hash family, and trusted-setup
  status. A circuit author working on Orchard must not import
  Sapling personalization strings (or vice versa). The personalization
  is what binds a commitment to its pool; reusing it would let a
  Sapling note replay into the Orchard tree.
- **Underrange nullifier scalar.** The scalar
  $\mathrm{PRF}^{\mathsf{nfOrchard}}_{\mathsf{nk}}(\rho) + \psi$ is reduced modulo
  the Pallas scalar field. Failing to range-check the addends
  in-circuit can produce two distinct $(\mathsf{nk}, \rho, \psi)$
  tuples with the same scalar, which collides nullifiers and
  enables double-spends. This is the canonical class of bug that
  audits target.
- **Reusing $\rho$ across actions.** $\rho$ is supposed to be
  derived from the *previous* action's nullifier, making the
  nullifier chain injective. A circuit that does not enforce
  the derivation rule lets a malicious prover create two notes
  with the same $\rho$ and therefore the same nullifier.
- **Wrong Merkle depth.** Sapling uses depth 32, Orchard uses
  depth 32. Sprout used 29. A mismatched constant breaks anchor
  verification.

## 7. Spec pointers

Primary sources, in order of relevance to the gadgets in this
repository:

- **Zcash Protocol Specification** (the normative reference for
  Sprout, Sapling, and Orchard primitives):
  [zips.z.cash/protocol/protocol.pdf](https://zips.z.cash/protocol/protocol.pdf).
  Section 4 ("Abstract Protocol") and section 5 ("Concrete
  Protocol") are the load-bearing chapters. The Orchard
  "MerkleCRH" subsection is at
  [#orchardmerklecrh](https://zips.z.cash/protocol/protocol.pdf#orchardmerklecrh);
  Orchard nullifier construction is
  [#commitmentsandnullifiers](https://zips.z.cash/protocol/protocol.pdf#commitmentsandnullifiers).
- **ZIP 224 "Orchard Shielded Protocol"**:
  [zips.z.cash/zip-0224](https://zips.z.cash/zip-0224). The
  consensus rules and transaction format.
- **ZIP 225 "Version 5 Transaction Format"**:
  [zips.z.cash/zip-0225](https://zips.z.cash/zip-0225). The serialized
  form of Orchard transactions, including `ActionDescription`.
- **Zerocoin paper** (Miers, Garman, Green, Rubin; IEEE S&P 2013):
  [PDF](https://zerocoin.org/media/pdf/ZerocoinOakland.pdf).
  Historical precursor; halo2 does not implement Zerocoin.
- **Zerocash paper** (Ben-Sasson et al.; IEEE S&P 2014):
  [PDF](http://zerocash-project.org/media/pdf/zerocash-extended-20140518.pdf).
  The construction Sprout and its successors inherit.
- **Halo paper** (Bowe, Grigg, Hopwood; eprint 2019/1021):
  [eprint 2019/1021](https://eprint.iacr.org/2019/1021). The IPA
  + accumulator construction that lets Orchard avoid a trusted
  setup.
- **Sinsemilla design**:
  [Halo 2 Book "Sinsemilla"](https://zcash.github.io/halo2/design/gadgets/sinsemilla.html).
- **Orchard circuit reference**:
  [zcash/orchard `src/circuit.rs`](https://github.com/zcash/orchard/blob/main/src/circuit.rs)
  composes the gadgets from this repository into the action
  predicate.

## 8. Exercises

1. Take any Sinsemilla constant in
   [`halo2_gadgets/src/sinsemilla/merkle.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs)
   and find the corresponding line in the Zcash Protocol
   Specification (`#orchardmerklecrh`). Confirm they agree.
2. Open
   [zcash/orchard/src/circuit.rs](https://github.com/zcash/orchard/blob/main/src/circuit.rs)
   and identify which `halo2_gadgets` import is used to compute
   the nullifier. Trace it back to the chip in this repository.
3. From the diagram in section 5, name one Orchard primitive that
   *does not* have a chip in `halo2_gadgets`. Why is that?
   (Hint: out-of-circuit primitives.)
4. Read section 4.2 of the Zcash Protocol Spec and identify
   exactly which information about a spent note is *revealed* on
   chain (i.e. appears in clear in an `ActionDescription`). Use
   this to argue that the nullifier is unlinkable to the
   commitment for an observer without the spending key.

### Answers in the code

- Exercise 1: the `MERKLE_CRH_PERSONALIZATION` constant in
  [`halo2_gadgets/src/sinsemilla/merkle.rs#L16`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_gadgets/src/sinsemilla/merkle.rs#L16)
  matches the personalization in the spec's section
  "Sinsemilla hashes and commitments".
- Exercise 3: `DiversifyHash` is an out-of-circuit primitive
  (the receiver computes their own diversifier off-chain); only
  the resulting $\mathsf{g_d}$ point appears as a witness.

## 9. Further reading

- [Sean Bowe, "Explaining Halo 2"](https://electriccoin.co/blog/explaining-halo-2/) for
  the design history of why Orchard moved off Groth16.
- [Daira Hopwood's
  Orchard explainer](https://electriccoin.co/blog/explaining-orchard/) for a
  user-facing overview of the pool.
- The
  [Zcash Protocol Specification](https://zips.z.cash/protocol/protocol.pdf)
  itself; chapters 2-5 are the authoritative reference for every
  primitive this course mentions.
