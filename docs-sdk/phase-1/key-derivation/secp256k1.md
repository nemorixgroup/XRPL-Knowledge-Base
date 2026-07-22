# secp256k1 Key Derivation

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.4-dev`)  
**Implementation:** [`lib/src/crypto/xrpl_secp256k1.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_secp256k1.dart)

## What This Is

XRPL's original signing algorithm, the same scheme Bitcoin uses.
Deriving a usable key pair from a seed's entropy takes more steps
here than Ed25519, for two reasons documented directly on xrpl.org:

- Not every 32-byte number is a valid secp256k1 secret key.
- XRPL's reference implementation still carries an unfinished
  "key family" framework - only the value 0 is ever actually used
  for it in practice, but the derivation steps it requires are still
  mandatory.

## The Algorithm

```
seed entropy (16 bytes) + root sequence (4 bytes, starts at 0)
v SHA-512Half
candidate private key
v  invalid (zero, or >= group order)? increment sequence, retry
ROOT KEY PAIR (private key + public key point)
v compress root public key (33 bytes)
v
root public key (33) + zeros (4) + intermediate sequence (4 bytes, starts at 0)
v SHA-512Half
candidate private key
v  invalid? increment sequence, retry
INTERMEDIATE KEY PAIR
v
master private key = (root + intermediate) mod group order
master public key   = root point + intermediate point (EC addition)
v
MASTER KEY PAIR  <- the account's real key pair
```

## Decision Taken

Three public methods on `XrplSecp256k1`, mirroring the three stages
above, so each stage is independently testable and so
`deriveRootKeyPair` alone is enough for validator key generation
(validators use the root pair directly and stop there, per the
official spec):

```dart
static XrplSecp256k1KeyPair deriveRootKeyPair(XrplEntropy entropy)
static XrplSecp256k1KeyPair deriveIntermediateKeyPair(Uint8List rootPublicKey)
static XrplSecp256k1KeyPair deriveKeyPair(XrplEntropy entropy) // root + intermediate combined
```

Both `deriveRootKeyPair` and `deriveIntermediateKeyPair` share one
private "hash, validate, retry" loop (`_deriveValidKeyPair`) - the two
steps differ only in what bytes get hashed, not in the validation or
retry logic, so that logic exists in exactly one place.

## Why `pointycastle`'s `ECDomainParameters`, Not a Hardcoded Constant

The secp256k1 group order is a specific 256-bit constant
(`0xFFFF...FE BAAE DCE6 AF48 A03B BFD2 5E8C D036 4141`). We use
`pointycastle`'s `ECDomainParameters('secp256k1')` to get this value
(and the curve's generator point, for point multiplication/addition)
rather than transcribing that number by hand. Given a manual
transcription error we already caught once in this SDK's history (an
incorrect seed prefix, documented in
[family-seed-encoding](../family-seed-encoding/README.md)), letting a
well-known library own large security-critical constants was a
deliberate choice, not just convenience.

## Public Key Compression

`XrplSecp256k1KeyPair.compressedPublicKey` uses `pointycastle`'s
`ECPoint.getEncoded()` directly (compressed by default: a one-byte
prefix - `0x02` if the Y coordinate is even, `0x03` if odd - plus the
32-byte X coordinate), rather than implementing the even/odd check
and byte-packing manually.

## Test Vectors / Verification

Every stage is checked against a value computed independently via
Python's `ecdsa` library, chained end-to-end from the same real
secp256k1 seed entropy already verified in
[family-seed-encoding](../family-seed-encoding/README.md) (from the
official seed `sn259rEFXrQrWyx3Q7XneWcwV6dfL`):

| Stage | Verified against |
|-------|--------------------|
| Root key pair | Independent Python `ecdsa` computation |
| Intermediate key pair | Same, chained from the verified root public key |
| Master key pair | Same, chained from both verified stages (modular addition + EC point addition) |

Additionally, the universal secp256k1 test vector (private key `1`
derives the curve's generator point `G` itself) is used to sanity
check the public key derivation and compression logic independent of
any XRPL-specific input.

12 tests total across three test files. See
[`test/crypto/xrpl_secp256k1_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_secp256k1_test.dart),
[`..._intermediate_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_secp256k1_intermediate_test.dart),
[`..._master_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_secp256k1_master_test.dart).

## Official Sources

- [xrpl.org - Cryptographic Keys: secp256k1 Key Derivation](https://xrpl.org/docs/concepts/accounts/cryptographic-keys#secp256k1-key-derivation) - the authoritative step-by-step specification this implementation follows exactly
