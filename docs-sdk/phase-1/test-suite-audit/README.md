# Test Suite Audit

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.1.0-dev`)  

## What This Is

The second half of the Phase 1 closing audit. Where
[error-handling](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/error-handling/README.md) reviewed the source
code itself, this pass reviewed the **93 tests** built up incrementally
across `0.0.1-dev` through `0.0.5-dev`, looking for duplication,
coverage gaps, and organizational issues before closing the phase.

## Method

Every test file under `test/` was reviewed against:

1. **Duplication** - are any two tests verifying the same thing for
   no reason?
2. **Coverage gaps** - is there any input/output combination no file
   covers?
3. **Organization** - does the current file layout still make sense
   now that the SDK has grown?

## Inventory at Audit Time

```
test/
codec/
   xrpl_base58_test.dart (encodeRaw)
   xrpl_base58_checksum_test.dart (checksumOf, encodeWithChecksum)
   xrpl_base58_decode_test.dart (decodeRaw)
   xrpl_base58_decode_checksum_test.dart (decodeWithChecksum)
crypto/
   xrpl_entropy_test.dart
   xrpl_hash_test.dart
   xrpl_key_algorithm_test.dart
   xrpl_seed_test.dart
   xrpl_secp256k1_test.dart (root key pair)
   xrpl_secp256k1_intermediate_test.dart
   xrpl_secp256k1_master_test.dart
   xrpl_ed25519_test.dart
wallet/
   xrpl_wallet_test.dart
```

## Decision: File Fragmentation Was Left As-Is

`xrpl_base58` is split across 4 files (one per method) and
`xrpl_secp256k1` across 3 (one per derivation stage), rather than one
file per class as most of the rest of the suite follows. This was
deliberately **not** consolidated during the audit - the fragmentation
mirrors real implementation milestones (`encodeRaw` shipped before
`decodeRaw`, root before intermediate before master), and each file
already reads clearly on its own. Merging them would have been a
cosmetic change with no effect on coverage or clarity, so it was
skipped in favor of focusing the audit on things that actually change
behavior.

## Result: No Duplication Found

Several tests intentionally re-verify a value already checked in an
earlier file - for example,
`xrpl_secp256k1_intermediate_test.dart` re-checks `root.compressedPublicKey`
before deriving the intermediate pair from it, and `xrpl_wallet_test.dart`
reuses the same real seeds already verified in the `secp256k1` and
`Ed25519` test files. These are **intentional chain-verification
tests**, not accidental duplication: each confirms a prior step still
holds before building on top of it, and removing them would weaken
the guarantee that the full chain (not just each isolated piece)
works correctly.

## Result: 2 Coverage Gaps Found and Closed

Both gaps were about **error propagation across layers**, not missing
logic. Each layer had its own errors well-tested in isolation, but
nothing confirmed those errors actually reached the layer above it
unmodified.

### Gap 1: `XrplWallet.fromSeed` with a Corrupted Seed

`XrplSeed.fromBase58` had thorough invalid-input tests, but nothing
confirmed that calling `XrplWallet.fromSeed` (the method a real SDK
user actually calls) with the same bad input still produced a clear
`XrplCryptoException`, rather than an unhandled error somewhere in
between.

```dart
test('propagates XrplCryptoException for a corrupted/invalid seed',
    () async {
  expect(
    () => XrplWallet.fromSeed(
      'esto-no-es-un-seed-valido',
      algorithm: XrplKeyAlgorithm.secp256k1,
    ),
    throwsA(isA<XrplCryptoException>()),
  );
});
```

### Gap 2: `XrplSeed.fromBase58` with an Invalid Base58 Character

The inverse problem, one layer down: `XrplBase58.decodeRaw` and
`decodeWithChecksum` both had direct tests for an invalid character,
and the new `XrplWallet` test above proves the error reaches the top
- but nothing isolated the **middle** layer (`XrplSeed.fromBase58`)
on its own. Without this, a future change that accidentally swallowed
the error inside `XrplSeed` specifically could pass every other test
in the suite (the `XrplWallet` test happens not to isolate which
layer produced the exception) while still being broken.

```dart
test('propagates an invalid-character error from XrplBase58.decodeRaw',
    () {
  expect(
    () => XrplSeed.fromBase58('s0notavalidseed'),
    throwsA(isA<XrplCryptoException>()),
  );
});
```

### Why Both Were Needed

```
XrplBase58.decodeRaw -> tested directly
XrplBase58.decodeWithChecksum -> tested directly (propagates from decodeRaw)
XrplSeed.fromBase58 -> Gap 2 (was missing, now tested directly)
XrplWallet.fromSeed -> Gap 1 (was missing, now tested directly)
```

Each layer can now fail "on its own" in the test suite, rather than
some layers only inheriting confidence from the layer below them.

## Final Count

**93 tests** (up from 91 before this audit), across 13 files.

## Status

This closes the full Phase 1 audit
([error-handling](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/error-handling/README.md) +
test-suite-audit). Phase 1 is complete as of `0.1.0-dev`.

## Related

- [Error Handling & Validation Audit](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/error-handling/README.md)
- [XrplWallet Unified API](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/wallet/README.md)
