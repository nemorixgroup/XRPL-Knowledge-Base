# Entropy Generation

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.1-dev`)  
**Implementation:** [`lib/src/crypto/xrpl_entropy.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_entropy.dart)  

## What This Is

Entropy is the random foundation every XRPL seed is built from. Before
a seed, a key pair, or an address can exist, we need 16 bytes of
cryptographically secure randomness.

## Decision Taken

Implemented `XrplEntropy`, a value type wrapping exactly 16 bytes of
randomness, generated via Dart's `dart:math` `Random.secure()`.

```dart
factory XrplEntropy.generate() {
  final random = Random.secure();
  final generated = Uint8List(lengthInBytes); // 16
  for (var i = 0; i < lengthInBytes; i++) {
    generated[i] = random.nextInt(256);
  }
  return XrplEntropy._(generated);
}
```

## Why 16 Bytes

The official `xrpl-keypairs` library (Ripple's reference
implementation) specifies that entropy, when provided explicitly,
"must be 16 bytes long." This is not an SDK-specific choice - it's a
hard requirement of the XRPL family seed format itself. Providing a
different length would produce a seed incompatible with the rest of
the XRPL ecosystem (wallets, explorers, other SDKs).

## Why `dart:math` Instead of `dart:io`

`Random.secure()` is available from `dart:math`, which works
identically on every Flutter platform, including Web. `dart:io`, by
contrast, does not exist in a browser context at all - any dependency
on it breaks Flutter Web compilation entirely.

The SDK does not support Web yet (see the main README for the current
platform list), but choosing `dart:math` now means Phase 1
cryptography code will not need to be rewritten if/when Web support is
added later - it's a zero-cost decision made early.

## Why a Value Type, Not a Bare Function

An earlier, simpler design considered a single static function
returning `Uint8List`. We chose a small wrapper type (`XrplEntropy`)
instead, because entropy needs to flow through several later steps
(seed encoding in `0.0.3-dev`, key derivation in `0.0.4-dev` and
`0.0.5-dev`) and needs the same validation (`fromBytes`, `validate`)
whether it was just generated or restored from storage.

```dart
final fresh = XrplEntropy.generate();
final restored = XrplEntropy.fromBytes(previouslySavedBytes);
```

## Error Handling

Invalid-length input never fails silently. `XrplEntropy.fromBytes()`
and `XrplEntropy.validate()` throw a dedicated `XrplCryptoException`
with the expected and actual lengths in the message, rather than
producing malformed entropy that could propagate into a broken seed
further down the pipeline.

## Test Vectors / Verification

Since entropy is randomly generated (there is no fixed "expected
output" the way there is for encoding functions), verification
focuses on:

- **Length invariant:** every generated or restored `XrplEntropy` is
  exactly 16 bytes, enforced by `validate()`
- **Randomness:** two consecutive `generate()` calls never produce the
  same bytes
- **Non-triviality:** generated entropy is not all-zero
- **Immutability:** `fromBytes()` does not share a mutable reference
  with the caller's original list

11 unit tests cover these properties. See
[`test/crypto/xrpl_entropy_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_entropy_test.dart).

## Official Sources

- [`xrpl-keypairs` (npm)](https://www.npmjs.com/package/xrpl-keypairs) - entropy length specification
- [xrpl.org - Cryptographic Keys](https://xrpl.org/docs/concepts/accounts/cryptographic-keys) - family seed background
