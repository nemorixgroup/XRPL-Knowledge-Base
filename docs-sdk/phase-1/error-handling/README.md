# Error Handling & Validation Audit

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.6-dev`)  

## What This Is

Unlike every other file in `docs-sdk/`, this one doesn't document a
new feature - it documents an **audit pass** over everything already
built in Phase 1 (`0.0.1-dev` through `0.0.5-dev`). By the time this
sub-version started, error handling had already been built
incrementally into each prior file rather than left for the end, so
the goal here was to verify that was actually true, not to add a
large amount of new validation.

## Method

Every source file under `lib/src/` was reviewed against three
questions:

1. **Missing edge cases** - is there any invalid input the code
   doesn't explicitly reject?
2. **Clear error messages** - does every `XrplCryptoException`
   explain what went wrong, without making the caller guess?
3. **Documentation accuracy** - does every doc comment still
   correctly describe what the code does?

## Result Summary

| File | Outcome |
|------|---------|
| `xrpl_entropy.dart` | ✅ Audited, no changes needed |
| `xrpl_base58.dart` | 1 fix: incorrect doc comment |
| `xrpl_seed.dart` | ✅ Audited, no changes needed |
| `xrpl_secp256k1.dart` | 1 new validation + 2 outdated-doc fixes |
| `xrpl_ed25519.dart` | 1 outdated-doc fix (surfaced an import cycle) |
| `xrpl_wallet.dart` | ✅ Audited, no changes needed |

Three files passed with no changes at all - confirmation that
building validation incrementally, file by file, worked as intended
rather than leaving gaps for an end-of-phase cleanup to find.

## Fix 1: Incorrect Doc Comment (`xrpl_base58.dart`)

The private field `_base` (the numeric base, `58`, used in the
big-integer conversion) had a doc comment copy-pasted from the
`alphabet` field above it:

```dart
// Was describing the wrong thing entirely:
/// The XRPL base58 alphabet (58 characters, no `0`, `O`, `I`, or `l`).
static final BigInt _base = BigInt.from(58);
```

Fixed to describe what the field actually is. A reminder that
documentation errors are still errors, even when the underlying logic
is correct - copy-pasted doc comments are an easy way for wrong
information to end up next to right code.

## Fix 2: Missing Input Validation (`xrpl_secp256k1.dart`)

`deriveIntermediateKeyPair(Uint8List rootPublicKey)` is a public
method, but never validated that `rootPublicKey` was exactly 33 bytes
(a compressed secp256k1 public key). Internally, the SDK always calls
it correctly, so this never surfaced in our own tests - but an
external caller passing the wrong length would have gotten a
key pair back with no error, one that simply wouldn't match what any
other XRPL tool would derive from the same seed.

```dart
const expectedLength = 33;
if (rootPublicKey.length != expectedLength) {
  throw XrplCryptoException(
    'rootPublicKey must be exactly $expectedLength bytes '
    '(a compressed secp256k1 public key), got '
    '${rootPublicKey.length}',
  );
}
```

This is the one genuinely new piece of validation added during this
audit, consistent with the SDK-wide rule: never accept malformed
input and produce output that merely *looks* valid.

## Fix 3 & 4: Outdated Documentation (`xrpl_secp256k1.dart`, `xrpl_ed25519.dart`)

Two doc comments in `xrpl_secp256k1.dart` still said combining the
root and intermediate key pairs was *"not yet implemented in this
sub-version"* - true when first written, false by the time
`deriveKeyPair` was added in the same sub-version, and never updated.
Similarly, `xrpl_ed25519.dart`'s class doc still said `XrplWallet`
*"will expose"* a unified async API - true before `0.0.5-dev`, stale
after `XrplWallet` actually shipped.

Both were corrected to describe the current, real state of the code
rather than a past-tense plan.

## A Near-Miss: an Import Cycle from a Documentation Fix

While correcting the `xrpl_ed25519.dart` doc comment, `` `XrplWallet` ``
(plain text) was changed to `[XrplWallet]` (a real dartdoc reference),
which required actually importing `xrpl_wallet.dart`. That created a
real, if easy to miss, structural problem:

xrpl_ed25519.dart --imports--> xrpl_wallet.dart
^ |
+-------------imports------------+

`xrpl_wallet.dart` already imports `xrpl_ed25519.dart` (it needs it to
derive keys) - so this made a low-level cryptographic primitive
depend on the higher-level layer that consumes it, backwards from how
the architecture is meant to flow. Dart doesn't reject import cycles
the way some languages do, so nothing failed loudly; only a
transitive-import warning from the analyzer (caused by an unrelated
mistake made while fixing this) surfaced it at all.

**Resolution:** reverted to a plain-text mention (`` `XrplWallet` ``,
no brackets, no import) rather than a real dartdoc cross-reference.
The lesson generalizes: a "lower" module in this SDK's layering
(`crypto/`, `codec/`) should never import from a "higher" one
(`wallet/`), even just for a documentation link.

## Status

Audit complete for `lib/src/`.  
Test suite consolidation is documented
separately in
[`docs-sdk/phase-1/test-suite-audit/`](../test-suite-audit/README.md).
