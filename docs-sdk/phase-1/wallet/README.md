# XrplWallet - Unified Public API

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.5-dev`)  
**Implementation:** [`lib/src/wallet/xrpl_wallet.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/wallet/xrpl_wallet.dart)  

## What This Is

Everything built so far in Phase 1 - entropy, the base58 codec,
family seed encoding, and key derivation for both signing algorithms
- exists as separate, independently testable pieces. `XrplWallet` is
the layer that ties them together into the single API most SDK users
will actually reach for: generate or restore a wallet, get a key
pair, done.

## Decision Taken

```dart
static Future<XrplWallet> generate({required XrplKeyAlgorithm algorithm})
static Future<XrplWallet> fromSeed(String value, {required XrplKeyAlgorithm algorithm})
```

Two factory-style static methods, both requiring `algorithm`
explicitly - consistent with every other type in this SDK, where the
signing algorithm is never inferred silently.

## Why `lib/src/wallet/`, Not `lib/src/crypto/wallet/`

`wallet/` was placed as a **sibling** to `crypto/`, `codec/`, and
`exceptions/`, not nested inside `crypto/`. A wallet is not a
cryptographic primitive - it's the layer that *combines* primitives
(`XrplSeed`, `XrplSecp256k1`, `XrplEd25519`) into something usable.
Nesting it under `crypto/` would have implied it belonged to that
category rather than sitting above it. This also anticipates Phase 2:
`XrplWallet` will likely grow to include the derived address too,
mixing concerns from `crypto/` and a future addresses module -
another reason it shouldn't be filed as "a kind of crypto."

## Why Public/Private Keys Are Unified `Uint8List`, Not Algorithm-Specific Types

Internally, `secp256k1` represents its private key as a `BigInt`
(it's a scalar used in curve arithmetic) while `Ed25519` represents
its private key as a `Uint8List` (it's used directly as bytes). Left
as-is, `XrplWallet` would have to expose two different shapes
depending on which algorithm was used.

We considered two options:

1. **Unified**: always expose `Uint8List privateKeyBytes` /
   `Uint8List publicKeyBytes`, converting `secp256k1`'s `BigInt` to
   bytes internally.
2. **Original types preserved**: expose both `BigInt?
   secp256k1PrivateKey` and `Uint8List? ed25519PrivateKeyRaw`, nullable
   depending on which algorithm was used.

**We chose unified (option 1)** and deliberately did **not** also
expose the original per-algorithm types alongside it. Nullable,
algorithm-specific fields would have reintroduced the exact kind of
"which field do I use, and why is the other one null?" confusion that
unifying was meant to remove. If a concrete need for the original
`BigInt` form surfaces later, it can be added then, with real
justification, rather than speculatively now.

Converting `BigInt` to bytes is a single, already-battle-tested
helper (`_bigIntToBytes`), reused from the same logic already applied
throughout the `secp256k1` test suite.

## Why the Public API Is Uniformly Asynchronous

`secp256k1` derivation (via `pointycastle`) is genuinely synchronous.
`Ed25519` derivation (via `package:cryptography`) is genuinely
asynchronous - and, per the analysis in
[key-derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/README.md), can't be made
synchronous without either an unmaintained dependency or
reimplementing curve math ourselves.

Rather than exposing that split to SDK users (a sync method for one
algorithm, an async method for the other), `XrplWallet.generate()` and
`XrplWallet.fromSeed()` are always `Future`-returning, regardless of
algorithm. Internally, the `secp256k1` path is just wrapped in an
`async` function - no real asynchronous work happens, but the public
shape stays identical either way:

```dart
static Future<XrplWallet> _deriveFrom(XrplSeed seed, XrplKeyAlgorithm algorithm) async {
  if (algorithm == XrplKeyAlgorithm.secp256k1) {
    final keyPair = XrplSecp256k1.deriveKeyPair(seed.entropy); // sync internally
    return XrplWallet._(...);
  }
  final keyPair = await XrplEd25519.deriveKeyPair(seed.entropy); // genuinely async
  return XrplWallet._(...);
}
```

## Closing a Validation Gap from `0.0.3-dev`

When `XrplSeed` added support for the `sEd...` prefix (which
explicitly declares Ed25519), we identified but deferred a related
risk: nothing yet stopped a developer from decoding an
Ed25519-declared seed and deriving a `secp256k1` key pair from it
anyway - silently producing a key pair the seed never intended.

`XrplWallet.fromSeed()` is the natural place to close this, since it's
the first point where a decoded seed and a requested algorithm meet:

```dart
if (seed.declaredAlgorithm != null && seed.declaredAlgorithm != algorithm) {
  throw XrplCryptoException(
    'Seed declares algorithm ${seed.declaredAlgorithm}, but '
    '$algorithm was requested',
  );
}
```

A seed with the generic `0x21` prefix (`declaredAlgorithm == null`)
makes no such declaration, so it can be used with either algorithm
without error - that's expected behavior, not a gap.

## Test Vectors / Verification

Beyond structural tests (key lengths, determinism, round-trips,
mismatch rejection), two full vectors are verified **end-to-end
through `XrplWallet` itself**, not just through its underlying
building blocks - reusing the same official seeds already verified in
earlier sub-versions:

| Algorithm | Seed | Verified against |
|-----------|------|--------------------|
| secp256k1 | `sn259rEFXrQrWyx3Q7XneWcwV6dfL` | The master key pair vector from `xrpl_secp256k1_master_test.dart` |
| Ed25519 | `sEdTM1uX8pu2do5XvTnutH6HsouMaM2` | The key pair vector from `xrpl_ed25519_test.dart` (originally from `pynacl`) |

14 tests total. See
[`test/wallet/xrpl_wallet_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/wallet/xrpl_wallet_test.dart).

## Related

- [Family Seed Encoding](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/family-seed-encoding/README.md)
- [Key Derivation (secp256k1 + Ed25519)](https://github.com/nemorixgroup/XRPL-Knowledge-Base/tree/main/docs-sdk/phase-1/key-derivation#readme)
