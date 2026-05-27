---
sidebar_position: 12
title: Fiat-Shamir Transcripts
description: How challenges are squeezed, the Blake2b transcript implementation, and the difference between common_point and write_point.
---

# Fiat-Shamir Transcripts

## 1. Why this chapter exists

Every challenge in halo2 ($\theta$, $\beta$, $\gamma$, $y$, $x$,
the IPA round challenges, the multiopen $x_1 \dots x_4$) comes
from a single Fiat-Shamir transcript. The transcript is what
makes the protocol non-interactive; it is also the part of the
codebase where a single off-by-one operation silently breaks
soundness. This chapter covers the trait layer (so you can write
a custom transcript), the Blake2b implementation (so you know
exactly what halo2 hashes), and the `common_*` vs `write_*`
distinction (which is the most common source of transcript bugs).

## 2. Definitions

**Definition (Transcript).** A stateful object that mirrors the
view of one party of the protocol. It absorbs the messages sent
by the prover and emits challenges when asked, deterministically
as a function of everything absorbed so far.

**Definition (Common input).** A value that is part of the
shared protocol input (the verifying key, the instance) and is
*not* sent over the wire as part of the proof. Both prover and
verifier hash it into their transcript independently.

**Definition (Prover-to-verifier message).** A value the prover
sends and the verifier reads. Both sides hash it into the
transcript; the prover *additionally* writes it to the proof
byte stream, and the verifier *additionally* reads it from there.

**Definition (Challenge).** A value squeezed out of the
transcript. In halo2 a raw challenge is a 32-byte uniformly
random string; a `ChallengeScalar<C, T>` is a typed scalar
derived from it.

## 3. The code

### 3.1 The trait layer

```rust file=../../halo2_proofs/src/transcript.rs#L22-L65 title="halo2_proofs/src/transcript.rs"
```
Three traits cleanly separate concerns:

- `Transcript`: the shared base. Includes `squeeze_challenge`,
  `squeeze_challenge_scalar`, `common_point`, `common_scalar`.
- `TranscriptRead` (verifier side): adds `read_point`,
  `read_scalar`. Each of these hashes the value into the
  transcript *and* returns it.
- `TranscriptWrite` (prover side): adds `write_point`,
  `write_scalar`. Each of these hashes the value into the
  transcript *and* writes it to the byte stream.

The `common_*` vs `write_*` distinction:

- `common_*` is used for values both sides already know (the VK
  itself, the public instance). Both sides call `common_*`;
  nothing crosses the wire.
- `write_*` is used for values the prover is *sending*. Only the
  prover calls `write_*`; the verifier calls `read_*` to consume
  the bytes and absorb the same value.

If the prover calls `write_*` where it should call `common_*`,
the verifier will see the value twice in its transcript hash and
the challenges will diverge. The reverse is also a soundness
break.

### 3.2 The Blake2b implementation

```rust file=../../halo2_proofs/src/transcript.rs#L64-L160 title="halo2_proofs/src/transcript.rs"
```
The `Blake2bRead` and `Blake2bWrite` types are the production
default. Three implementation choices to remember:

- The hash is Blake2b-512 (64-byte output), personalized with
  `"Halo2-Transcript"`.
- A 1-byte prefix is hashed before every absorbed value:
  `BLAKE2B_PREFIX_CHALLENGE = 0`, `BLAKE2B_PREFIX_POINT = 1`,
  `BLAKE2B_PREFIX_SCALAR = 2`. These prevent domain confusion
  attacks (you cannot make a point hash collide with a scalar
  hash).
- The `ChallengeScalar<C, T>` is squeezed by reducing the 64-byte
  Blake2b output modulo the scalar field order, via
  `FromUniformBytes`. This gives statistical uniformity in the
  scalar field.

### 3.3 Typed challenges

```rust file=../../halo2_proofs/src/transcript.rs#L230-L260 title="halo2_proofs/src/transcript.rs"
```
`ChallengeScalar<C, T>` carries a phantom type `T` so the
compiler can distinguish "the lookup challenge $\theta$" from
"the permutation challenge $\beta$" even though both are scalars.
Each user of the transcript defines a marker type (the multiopen
types `X1`, `X2`, `X3`, `X4` are an example):

```rust
#[derive(Clone, Copy, Debug)]
struct X1 {}
type ChallengeX1<F> = ChallengeScalar<F, X1>;
```

This catches "I asked for $x_2$ but used it as if it were $x_3$"
at compile time.

### 3.4 The `Challenge255` encoding

```rust file=../../halo2_proofs/src/transcript.rs#L270-L305 title="halo2_proofs/src/transcript.rs"
```
`Challenge255` is the in-protocol challenge representation: a
255-bit random string (32 bytes with the top bit masked off).
Most halo2 code never sees it directly; it appears inside the
`EncodedChallenge` trait that abstracts over future challenge
encodings.

## 4. Failure modes

- **`common_*` / `write_*` confusion.** Swapping one for the
  other silently breaks soundness. There are no compile-time
  checks; review every transcript call when changing the prover.
- **Re-squeezing without absorbing.** Two consecutive
  `squeeze_challenge` calls return *different* values (Blake2b is
  stateful), but only because the state is updated by the
  squeeze. Calling `squeeze_challenge` twice without absorbing
  anything in between is rarely what you want; the resulting
  challenges are still random but they correspond to different
  protocol steps. If both prover and verifier do the same thing
  this is safe, but it almost certainly indicates a bug.
- **Hashing the VK twice.** The VK should be hashed exactly once
  at the start of the proof, via
  `vk.hash_into(transcript)`. Doing it twice (or not at all)
  silently changes every subsequent challenge.
- **Custom transcripts forgetting the domain prefixes.** A
  custom `Transcript` implementation must keep separate domains
  for points, scalars, and challenges, or it admits a
  point-as-scalar confusion attack.

## 5. Spec pointers

- The Fiat-Shamir heuristic is treated formally in
  [Bellare and Rogaway, "Random oracles are practical",
  CCS 1993](https://www.cs.ucdavis.edu/~rogaway/papers/ro.pdf).
  Modern security proofs in the algebraic group model live in
  [Fuchsbauer et al., "ROM-AGM"](https://eprint.iacr.org/2020/1213).
- [BLAKE2: simpler, smaller, fast as MD5](https://www.blake2.net/blake2.pdf)
  for the underlying hash construction.

## 6. Exercises

1. Open
   [`halo2_proofs/src/transcript.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/transcript.rs)
   and identify the three `BLAKE2B_PREFIX_*` constants. What
   would happen if you set two of them to the same value?
2. Implement a `NoopTranscript` that records the operations
   performed against it (without computing anything) and write a
   small test that compares the prover's call sequence to the
   verifier's. Confirm they agree token-by-token.
3. Find the `hash_into` method on `VerifyingKey` in
   [`halo2_proofs/src/plonk.rs`](https://github.com/zcash/halo2/blob/32a87582dfb0ad9364ef3ffe71751ceab2a502ea/halo2_proofs/src/plonk.rs)
   and trace exactly which fields of the VK are absorbed.
4. Why does `ChallengeScalar<C, T>` carry a phantom `T`? Answer
   in one sentence.

### Answers in the code

- Exercise 1: the constants are 0, 1, 2. Aliasing them would
  remove the domain separation between challenges, points, and
  scalars, opening the transcript to a confusion attack.
- Exercise 3: `VerifyingKey::hash_into` calls
  `transcript.common_scalar(self.transcript_repr)`, where
  `transcript_repr` is itself the Blake2b-512 hash of the
  pinned verifying key (see the `from_parts` method).
- Exercise 4: the phantom binds each challenge to a particular
  protocol step at compile time, preventing accidental reuse.

## 7. Further reading

- The
  [Halo 2 Book "Implementation" chapter](https://zcash.github.io/halo2/design/implementation.html)
  describes the transcript layer in the context of the full
  proving system.
