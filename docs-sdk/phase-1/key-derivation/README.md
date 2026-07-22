# Key Derivation (secp256k1 + Ed25519)

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.4-dev`)  
**Implementation:** [`lib/src/crypto/xrpl_secp256k1.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_secp256k1.dart), [`lib/src/crypto/xrpl_ed25519.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_ed25519.dart), [`lib/src/crypto/xrpl_hash.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_hash.dart)

## What This Is

Turning a seed (`0.0.3-dev`) into a real, usable key pair - a private
key that can sign transactions, and a public key that the network can
use to verify those signatures. XRPL supports two independent signing
algorithms, and both share this folder because they're two answers to
the same question rather than two separate features.

## Why Both Algorithms Are Documented Together

- [secp256k1.md](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/secp256k1.md) - the original, default XRPL algorithm
- [ed25519.md](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/ed25519.md) - the newer, faster algorithm many client
  libraries default to

They share one hashing primitive (`XrplHash.sha512Half`), and the
single most important decision in this sub-version applies across
both: **the two derivation methods have different API shapes**, and
that difference is deliberate.

## The Sync vs. Async Decision

| | secp256k1 | Ed25519 |
|---|---|---|
| Library | `pointycastle` | `package:cryptography` |
| API | Synchronous | Asynchronous (`Future`) |
| Why | `pointycastle` has native secp256k1 curve support | `pointycastle` has no Ed25519 signer at all (confirmed during `0.0.3-dev` planning) |

We considered three ways to make this consistent:

1. **Make secp256k1 async too**, by switching it to `package:cryptography`.
   Not possible: that library only supports Ed25519, X25519, and the
   NIST curves (P-256/P-384/P-521) - it has no secp256k1 support at
   all, confirmed directly against its API docs.
2. **Make Ed25519 sync**, by switching to a synchronous pure-Dart
   package (`ed25519_edwards`). Rejected: that package has had no
   real updates in roughly four years. For cryptographic code, an
   unmaintained dependency is a real risk (an undiscovered bug or
   issue has no one to fix it), and the official XRPL guidance is
   explicit: *"use a standard, well-known, publicly-audited
   implementation whenever possible."*
3. **Implement Ed25519 ourselves** on top of a lower-level curve
   library (`edwards25519`, actively maintained but curve-math only).
   Rejected for the same reason: this SDK does not roll its own
   cryptographic primitives when a maintained, audited implementation
   exists.

**Decision:** keep the two algorithms on the libraries best suited to
each (correctness and maintenance first), and hide the inconsistency
one layer up. `XrplWallet` (`0.0.5-dev`) will expose a single,
uniformly asynchronous public API regardless of which algorithm is
requested - wrapping secp256k1's already-synchronous path in an
`async` function, rather than reimplementing anything.

## Verification Approach

Every derivation step in both files is checked against a value
independently computed outside this codebase - via Python's
`hashlib`, the `ecdsa` library (for secp256k1), and `pynacl`/libsodium
(for Ed25519) - not only against our own round-trip tests. See each
algorithm's page for the specific vectors used.

## Related

- [Family Seed Encoding](../family-seed-encoding/README.md) - produces the entropy these derivations consume  
- [secp256k1 Key Derivation]([secp256k1.md](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/secp256k1.md))  
- [Ed25519 Key Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/ed25519.md)  
