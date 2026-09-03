# Transaction Signing

**Phase:** 4 - Core Transactions  
**Status:** ✅ Done (`0.3.2-dev`)  
**Implementation:** [`lib/src/transactions/binary/`](https://github.com/nemorixgroup/xrpl-flutter-sdk/tree/main/lib/src/transactions/binary), [`lib/src/transactions/xrpl_signer.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_signer.dart), [`lib/src/crypto/xrpl_secp256k1.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_secp256k1.dart), [`lib/src/crypto/xrpl_ed25519.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/crypto/xrpl_ed25519.dart)

## What This Is

`0.3.1-dev` produced a complete, correctly-filled-in transaction.
This sub-version turns that into a cryptographically signed one,
provably authorized by a specific account. It's the largest and most
technically involved piece of work in this SDK so far, spanning five
new files and extending two Phase 1 modules with genuinely new
capability (signing, not just key derivation).

## Why JSON can't be signed directly

JSON has no single canonical representation - the same data can be
written with different spacing, field order, or number formatting.
Signing something that ambiguous would mean two people with the
identical *intent* could produce different signable bytes. XRPL
solves this with a **canonical binary format**: a fixed, unambiguous
byte-for-byte representation, built in five official steps (ensure
required fields are present, convert each field to its internal
binary format, sort fields in canonical order, prefix each with a
Field ID, concatenate).

## Verifying against a real official example, Byte by Byte

Before writing any serialization code, the official worked example
(an `OfferCreate` transaction's published JSON and binary side by
side) was decoded field-by-field in Python. Every single byte of the
official 218-byte hex blob was accounted for - nothing left over,
nothing missing. This gave concrete, confirmed values for every field
type this SDK needed (`UInt16`, `UInt32`, `AccountID`, `Blob`, both
`Amount` sub-types) before any Dart was written, rather than
implementing from the specification text alone and hoping it matched.

## Field Definitions: generated from a live server, not transcribed

### The problem

Every field's binary properties (its Field ID, its type) come from
a `definitions.json` reference file or an equivalent live API. Hand
-transcribing dozens of values from a webpage is exactly the kind of
error this SDK has been burned by before (the Phase 1 Ed25519 prefix
bug).

### The decision

Rather than transcribing by hand, or fetching definitions at runtime
(which would mean signing a transaction requires network access - not
acceptable for a wallet SDK), a **hybrid** approach was used:

1. `scripts/regenerate_field_definitions.dart` (an internal
   maintenance tool, not part of the public API) queries a real
   server's `server_definitions` command
2. Its output is reviewed, cross-checked against 7 independently
   confirmed known-good values, and manually committed into
   `XrplFieldDefinitions` as static Dart constants
3. The SDK itself never calls `server_definitions` at runtime -
   signing works fully offline

This mirrors how `xrpl.js` and `xrpl-py` maintain their own
definitions files: updated deliberately by maintainers when needed,
not fetched live by every application using the library.

### Why these values are safe to treat as permanent

A Field ID is part of how a transaction's signing hash is calculated.
Changing one for a field that already exists would invalidate every
historical transaction and signature on the XRP Ledger since 2012 -
in practice, impossible for Ripple to do without breaking every
client library and every past transaction simultaneously. Adding
fields for new transaction types (Phase 5 onward) is normal; changing
existing ones is not.

## A Real Bug Caught: canonical order is not byte order

**Caution** (directly from the official specification): *"you should
not sort by the serialized Field ID itself, because the byte
structure of the Field ID changes the sort order."*

This was confirmed concretely with the official example: `Expiration`
has a 1-byte Field ID (`0x2A`); `OfferSequence` has a 2-byte Field ID
(`[0x20, 0x19]`). Comparing the encoded bytes directly would rank
`OfferSequence` first (`0x20 < 0x2A`), but the real binary has
`Expiration` first - because canonical order compares `(typeCode,
fieldCode)` as separate numbers (`(2, 10)` vs. `(2, 25)`), which the
encoded byte lengths obscure. `XrplFieldDefinition` stores `typeCode`
and `fieldCode` explicitly, not only the encoded bytes, specifically
because of this.

## A Real Bug Caught: 16 significant digits, not 15

Issued-currency amounts (`LimitAmount`, and any future issued-currency
`Amount`) are normalized to a mantissa in the range `10^15` to
`10^16 - 1`. An initial implementation assumed 15 significant digits;
cross-checking the normalization algorithm against the official
`TakerPays` value (`7072.8` USD) surfaced the error immediately - 15
digits produced a mantissa/exponent pair that reconstructed to
`999.999...` instead of `7072.8`. Corrected to 16 digits, confirmed
exact against the official hex.

## `secp256k1` Signing: deterministic and fully canonical

XRPL requires **"fully canonical"** `secp256k1` signatures (enforced
protocol-wide since 2020): proper DER encoding, and - critically - the
numerically smaller of the two mathematically valid `S` values
(`low-S`). `XrplSecp256k1.sign`:

- Uses RFC 6979 deterministic nonce generation, via `pointycastle`'s
  `ECDSASigner(null, HMac(SHA256Digest(), 64))`. (Note: the Dart port
  of `pointycastle` does not have a class named `HMacDSAKCalculator`,
  despite that being the name in the original Java BouncyCastle
  library it's based on - the correct Dart equivalent, confirmed
  against a real, maintained third-party package's source, is passing
  an `HMac` instance directly.)
- Manually normalizes to `low-S` after signing, since a generic ECDSA
  signer does not enforce this on its own
- DER-encodes the result

### How this was verified

RFC 6979 guarantees a *given* implementation is deterministic with
itself - not that two independent implementations derive their nonce
identically and produce byte-identical signatures. An initial test
assumed byte-exact matching against Python's `ecdsa` library was the
right bar, and failed - not because the Dart signature was wrong, but
because that expectation was. The correct bar, and what this SDK
verifies, is **signature validity**: sign with Dart, verify with a
newly added `XrplSecp256k1.verify`, confirm it accepts the correct
message and rejects an incorrect one.

## `Ed25519` Signing: same hash, no canonicalization needed

Per the official specification, *"All valid Ed25519 signatures are
fully canonical"* - no `low-S`-style extra step needed, unlike
`secp256k1`.

### A detail worth confirming precisely: what gets signed

It was not obvious upfront whether XRPL feeds `Ed25519` the raw
serialized transaction (letting standard `EdDSA` do its own internal
hashing) or the same 32-byte `SHA-512Half` hash `secp256k1` signs.
Confirmed directly from the official Binary Format page: **both
algorithms sign the same hash** - the transaction is always hashed
with the appropriate prefix before either algorithm signs it.
`package:cryptography`'s `Ed25519.sign` still performs its own
standard internal hashing on whatever bytes it receives; XRPL's
specific choice is simply what that 32-byte "message" is.

### How this was verified

Unlike `secp256k1`, `Ed25519` signing is fully standardized and
deterministic (RFC 8032) with no implementation-specific nonce
variance - so this SDK's output was verified as an **exact
byte-for-byte match** against an independently computed signature
(Python's `pynacl`/libsodium), a stronger guarantee than the validity
check used for `secp256k1`.

## `sign()`: The complete pipeline

```dart
Future<Map<String, dynamic>> sign(
  Map<String, dynamic> transactionJson,
  XrplWallet wallet,
)
```

Ties every piece together: adds `SigningPubKey`, serializes via
`XrplTransactionSerializer`, prepends the single-signing prefix
(`0x53545800`, distinct from the multi-signing prefix
`0x534D5400`, which this SDK does not support yet), hashes with
`XrplHash.sha512Half` (Phase 1), signs with the algorithm-appropriate
method based on `wallet.algorithm`, and adds `TxnSignature` - without
mutating the input map.

Works on a plain `Map<String, dynamic>`, not the `XrplPayment`/
`XrplTrustSet` classes directly - the same design decision already
made for `XrplTransactionSerializer` in `0.3.1-dev`: signing needs a
`SigningPubKey` field and produces a `TxnSignature` field that don't
belong on the transaction model classes themselves.

## Two verified end-to-end scenarios

Rather than testing pieces in isolation only, `sign()` was verified
against two complete scenarios (one per algorithm), each built by
restoring a real `XrplWallet` from a seed **already fully verified in
Phase 1** (`sn259rEFXrQrWyx3Q7XneWcwV6dfL` for `secp256k1`,
`sEdTM1uX8pu2do5XvTnutH6HsouMaM2` for `Ed25519`) - so the test uses
real, previously-confirmed keys, not arbitrary ones. The expected
`SigningPubKey`, message hash, and (for `Ed25519`) exact signature
were independently computed via a standalone Python script before
being trusted.

## Test Vectors / Verification

70 tests across this sub-version's files, all traceable to either a
live `server_definitions` response (field definitions) or the
official `OfferCreate` worked example (primitives, amount encoding,
and the full serializer), plus two independently-computed end-to-end
signing scenarios. See
[`test/transactions/binary/`](https://github.com/nemorixgroup/xrpl-flutter-sdk/tree/main/test/transactions/binary),
[`test/crypto/xrpl_secp256k1_sign_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_secp256k1_sign_test.dart),
[`test/crypto/xrpl_ed25519_sign_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/crypto/xrpl_ed25519_sign_test.dart),
and
[`test/transactions/xrpl_signer_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/transactions/xrpl_signer_test.dart).

## Official Sources

- [xrpl.org - Binary Format](https://xrpl.org/docs/references/protocol/binary-format) - canonical serialization, Field IDs, canonical order, Amount encoding, the official OfferCreate worked example
- [xrpl.org - server_definitions](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/server-info-methods/server_definitions)
- [xrpl.org - Cryptographic Keys](https://xrpl.org/docs/concepts/accounts/cryptographic-keys) - signing algorithm requirements, canonical signatures
- [XRPL-Standards Discussion #79](https://github.com/XRPLF/XRPL-Standards/discussions/79) - confirms secp256k1 low-S canonicalization requirement and Ed25519's inherent canonicality

## Related

- [Transaction Model & Autofill](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/transaction-model/README.md) - produces the input `sign()` consumes
- [Family Seed Encoding](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/family-seed-encoding/README.md), [Key Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/key-derivation/README.md) - the seeds and keys this sub-version's verification chains from
