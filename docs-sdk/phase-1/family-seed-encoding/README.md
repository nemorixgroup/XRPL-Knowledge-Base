# Family Seed Encoding

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.3-dev`)  
**Implementation:** [`lib/src/crypto/xrpl_seed.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_seed.dart), [`lib/src/crypto/xrpl_key_algorithm.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_key_algorithm.dart)

## What This Is

A family seed is the human-shareable string (e.g. `sEdTNFV69uSpc...`)
a user saves to recreate their key pair and, later, their address.
It combines 16 bytes of entropy with a prefix that identifies it as
an XRPL seed, encoded with a checksum via the base58 codec from
`0.0.2-dev`.

## Decision Taken

Implemented `XrplSeed`, combining `XrplEntropy` (`0.0.1-dev`) and
`XrplBase58.encodeWithChecksum`/`decodeWithChecksum` (`0.0.2-dev`),
and a new `XrplKeyAlgorithm` enum (`secp256k1`, `ed25519`) shared
with the upcoming key derivation step.

```dart
factory XrplSeed.generate({required XrplKeyAlgorithm algorithm}) {
  return XrplSeed(
    XrplEntropy.generate(),
    declaredAlgorithm:
        algorithm == XrplKeyAlgorithm.ed25519 ? algorithm : null,
  );
}
```

## Supporting Both XRPL Seed Prefixes

XRPL defines two seed prefixes, per the official `ripple-address-codec`
reference implementation:

| Prefix | Bytes | Resulting string | Meaning |
|--------|-------|-------------------|---------|
| `FAMILY_SEED` | `0x21` (1 byte) | `s...` | Generic - does not declare an algorithm |
| `ED25519_SEED` | `0x01 0xE1 0x4B` (3 bytes) | `sEd...` | Explicitly declares Ed25519 |

We chose to support **both** on decode, and to encode with whichever
one matches the requested algorithm, rather than only using the
generic prefix. This is consistent with the official behavior
documented in `xrpl-py`: *"sEd… seeds derive an ED25519 keypair, all
other family seeds (s…) derive a SECP256K1 keypair."*

This matters beyond this SDK: a seed generated here may later be
imported into a different wallet or tool. If it doesn't self-declare
its algorithm, that other tool has to guess a default, which can
silently produce a *different* key pair (and address) than the one
the user expects - the same "same seed, different address" problem
discussed when we first decided the algorithm must always be an
explicit parameter, but now at the interoperability level rather than
just within our own API.

```dart
final seed = XrplSeed.generate(algorithm: XrplKeyAlgorithm.ed25519);
print(seed.toBase58()); // "sEdT..." - self-describing to any XRPL tool
```

Note there is no dedicated secp256k1 prefix - only Ed25519 has one -
so `declaredAlgorithm` is `null`, not an explicit enum value, for
secp256k1 seeds.

## A Bug We Caught: the Ed25519 Prefix Is 3 Bytes, Not 2

Our first implementation used a 2-byte Ed25519 prefix (`0x01 0xE1`).
This was based on manually decoding a real published seed with a
Python script and misreading where the prefix ended and the entropy
began - the third prefix byte (`0x4B`) was mistakenly attributed to
entropy instead.

The bug surfaced immediately when we tried to verify the
implementation against that same real seed: a freshly generated
Ed25519 seed didn't start with `sEd`, and decoding the known seed
threw an "unrecognized prefix" error instead of succeeding.

We then found the authoritative source - the `ripple-address-codec`
TypeScript source itself - which confirms:

```ts
const ED25519_SEED = [0x01, 0xE1, 0x4B] // [1, 225, 75]
```

This is the exact kind of error the SDK's "always verify against
official sources" rule exists to catch. It's documented here
deliberately, as a concrete example of why we verify against real,
independently-sourced test vectors rather than only our own
round-trip tests.

## Test Vectors / Verification

Four official test vectors from the `ripple-address-codec` test suite
(`xrp-codec.test.ts`) are used directly in our test suite, not just
vectors we computed ourselves:

| Entropy | Algorithm | Expected seed |
|---------|-----------|----------------|
| `4C3A1D213FBDFB14C7C28D609469B341` | Ed25519 | `sEdTM1uX8pu2do5XvTnutH6HsouMaM2` |
| all-zero (16 bytes) | Ed25519 | `sEdSJHS4oiAdz7w2X2ni1gFiqtbJHqE` |
| all-`0xFF` (16 bytes) | Ed25519 | `sEdV19BLfeQeKdEXyYA4NhjPJe6XBfG` |
| (decoded from) `sn259rEFXrQrWyx3Q7XneWcwV6dfL` | secp256k1 | (round-trip) |

All four were independently re-verified via a standalone Python
re-implementation of the encoding algorithm before being added to the
Dart test suite, specifically to avoid repeating the manual-decoding
mistake described above.

16 tests total for this feature. See
[`test/crypto/xrpl_seed_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_seed_test.dart).

## Error Handling

`XrplSeed.fromBase58()` throws `XrplCryptoException` when:

- the checksum doesn't match (via `XrplBase58.decodeWithChecksum`)
- the decoded data's length and prefix don't match either known
  format (neither 17 bytes starting with `0x21`, nor 19 bytes
  starting with `0x01 0xE1 0x4B`)

## Official Sources

- [`ripple-address-codec` - `xrp-codec.ts`](https://github.com/ripple/ripple-address-codec/blob/master/src/xrp-codec.ts) - authoritative prefix bytes and encode/decode logic
- [`ripple-address-codec` - `xrp-codec.test.ts`](https://github.com/ripple/ripple-address-codec/blob/master/src/xrp-codec.test.ts) - official test vectors used in our test suite
- [xrpl-py - Wallet documentation](https://xrpl-py.readthedocs.io/en/stable/source/xrpl.wallet.html) - confirms the `sEd...` vs generic prefix algorithm inference behavior
