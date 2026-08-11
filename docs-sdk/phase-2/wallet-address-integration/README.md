# Wallet Address Integration

**Phase:** 2 - Addresses  
**Status:** ✅ Done (`0.1.3-dev`)  
**Implementation:** [`lib/src/wallet/xrpl_wallet.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/wallet/xrpl_wallet.dart)

## What This Is

Address derivation (`XrplClassicAddress`, `XrplXAddress`) and key
derivation (`XrplWallet`) were built and verified as separate,
independent pieces on purpose, each with its own official test
vectors. This sub-version does not add any new cryptographic or
encoding logic - it wires those already-verified pieces together so
a caller gets a wallet's address without a separate call.

## Decision Taken

```dart
final String classicAddress; // field
String xAddress({required XrplNetwork network, int? tag}); // method
```

Two different shapes for two different reasons:

- `classicAddress` is a **field**, derived once inside `_deriveFrom`
  and cached, because it never changes for a given wallet - it's a
  pure function of the public key, and deriving it again on every
  access would only repeat work that already has an answer.
- `xAddress` is a **method**, because its result depends on
  parameters supplied at call time (`network`, and optionally `tag`).
  There's no single fixed value to cache - a wallet can have many
  valid X-addresses depending on which network and tag the caller
  asks for.

## Why Neither `XrplClassicAddress` nor `XrplXAddress` Changed

This sub-version only *calls* those types from inside `XrplWallet`;
their own implementation, validation, and test coverage from
`0.1.1-dev` and `0.1.2-dev` are unmodified. `XrplWallet`'s new tests
explicitly assert that `wallet.classicAddress` and `wallet.xAddress(...)`
produce output identical to calling `XrplClassicAddress.deriveFrom`
and `XrplXAddress.deriveFrom` directly with the same public key -
confirming the integration is a thin, faithful pass-through rather
than a second, parallel implementation that could drift from the
original.

## Test Vectors / Verification

No new official vectors were needed here - the underlying derivation
logic was already verified against official sources in
[Classic Address](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/classic-address/README.md) and
[X-Address](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/x-address/README.md). This sub-version's 9 tests focus
instead on confirming the integration itself: that both members are
present and correctly formatted for both algorithms, that they match
their standalone equivalents exactly, and that restoring a wallet
from its seed reproduces the same address.

See
[`test/wallet/xrpl_wallet_classic_address_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/wallet/xrpl_wallet_classic_address_test.dart)
and
[`test/wallet/xrpl_wallet_x_address_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/wallet/xrpl_wallet_x_address_test.dart).

## Related

- [Classic Address Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/classic-address/README.md)
- [X-Address Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/x-address/README.md)
- [XrplWallet Unified API](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/wallet/README.md) - the Phase 1 foundation this builds on
