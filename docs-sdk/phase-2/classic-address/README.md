# Classic Address Derivation

**Phase:** 2 - Addresses  
**Status:** ✅ Done (`0.1.1-dev`)  
**Implementation:** [`lib/src/address/xrpl_classic_address.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/address/xrpl_classic_address.dart)  

## What This Is

The classic address (`"r..."`) is the identifier most people
recognize as an XRPL account. It's derived one-way from a public key:
you can always confirm a public key matches an address, but never
recover the public key from the address alone.

## The Algorithm

```
Public key (33 bytes: compressed secp256k1, or Ed25519 with 0xED prefix)
    -> SHA-256
    -> RIPEMD-160
Account ID (20 bytes)
    -> prepend type prefix 0x00
21 bytes
    -> XrplBase58.encodeWithChecksum (double-SHA256 checksum)
Classic Address ("r...")
```

## Decision Taken

```dart
static Uint8List accountIdFromPublicKey(Uint8List publicKey)
static String deriveFrom(Uint8List publicKey)
```

Split into two methods deliberately: `accountIdFromPublicKey` exposes
the 20-byte Account ID on its own, since some XRPL contexts need the
raw Account ID rather than the encoded address (for example, some
ledger data structures reference accounts by ID directly). `deriveFrom`
is what most SDK users will actually call.

## Why This Reuses `XrplBase58.encodeWithChecksum`

Classic addresses use exactly the same Base58Check-style scheme
(XRPL's own alphabet, a 4-byte double-SHA256 checksum) already built
for seeds in Phase 1 - only the type prefix differs (`0x00` for
addresses vs. `0x21`/`0x01 0xE1 0x4B` for seeds). Rather than
duplicating that encoding logic, `XrplClassicAddress.deriveFrom` calls
straight into `XrplBase58.encodeWithChecksum`, the same function
`XrplSeed.toBase58()` already uses.

## Why `lib/src/address/` Is a New Sibling Folder

Following the same reasoning as `wallet/`: address derivation isn't a
cryptographic primitive (it doesn't generate or validate keys), it's
its own concept - all of Phase 2 is "Addresses." It also anticipates
where this is going: a later sub-version (`0.1.3-dev`) will integrate
this into `XrplWallet` directly, mixing concerns from `crypto/` and
`address/` the same way `wallet/` already mixes `crypto/` and `codec/`.

## Input Validation

`accountIdFromPublicKey` validates that the input is exactly 33
bytes - the length of every public key this SDK produces, whether
secp256k1 (compressed) or Ed25519 (with the `0xED` prefix). This
follows the same rule established during the Phase 1 audit
(`XrplSecp256k1.deriveIntermediateKeyPair`): a public method never
silently accepts a malformed input and produces output that merely
*looks* valid.

## Test Vectors / Verification

Verified against the **complete worked example published directly in
the official XRPL developer documentation**
(`xrpl-dev-portal/docs/concepts/accounts/addresses.md`), not a
third-party library - the exact public key, intermediate Account ID,
and final address xrpl.org itself uses to explain the algorithm:

| Value | |
|-------|--|
| Public key | `ED9434799226374926EDA3B54B1B461B4ABF7237962EAE18528FEA67595397FA32` |
| Account ID | `88a5a57c829f40f25ea83385bbde6c3d8b4ca082` |
| Address | `rDTXLQ7ZKZVKz33zJbHjgVShjsBnqMBhmN` |

Independently re-verified via a standalone Python script (SHA-256 +
RIPEMD-160 + base58 checksum, computed from scratch, not copied from
any library) before being added to the Dart test suite.

9 tests total. See
[`test/address/xrpl_classic_address_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/address/xrpl_classic_address_test.dart).

## Official Sources

- [xrpl.org - Addresses: Address Encoding](https://xrpl.org/docs/concepts/accounts/addresses#address-encoding) - the complete worked example this implementation matches exactly, including the C++ reference source (`AccountID.cpp`) it's derived from

## Related

- [XrplWallet Unified API](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/wallet/README.md) - the source of the public key bytes this consumes
