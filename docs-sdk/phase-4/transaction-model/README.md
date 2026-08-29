# Transaction Model & Autofill

**Phase:** 4 - Core Transactions  
**Status:** ✅ Done (`0.3.1-dev`)  
**Implementation:** [`lib/src/transactions/xrpl_transaction.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_transaction.dart), [`lib/src/transactions/models/`](https://github.com/nemorixgroup/xrpl-flutter-sdk/tree/main/lib/src/transactions/models), [`lib/src/transactions/xrpl_fee_strategy.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_fee_strategy.dart), [`lib/src/transactions/xrpl_autofill.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_autofill.dart)

## What This Is

Everything built through Phase 3 lets the SDK read from the XRP
Ledger. Phase 4 is where it can describe an intent to *change* the
ledger - starting with the two most common transaction types,
`Payment` and `TrustSet`. This sub-version only covers building a
complete, correctly-filled-in transaction; it is not yet signed
(`0.3.2-dev`) or submitted (`0.3.3-dev`).

## Decision Taken: A Shared `XrplTransaction` Interface

```dart
abstract class XrplTransaction {
  String get account;
  int? get sequence;
  String? get fee;
  int? get lastLedgerSequence;
  XrplTransaction copyWith({int? sequence, String? fee, int? lastLedgerSequence});
  Map<String, dynamic> toJson();
}
```

`XrplPayment` and `XrplTrustSet` both implement this. The alternative
considered was a separate `autofillPayment`/`autofillTrustSet` pair,
one per transaction type - rejected because Phase 4 is not the last
transaction type this SDK will add: `OfferCreate` (Phase 5),
`EscrowCreate` (Phase 6), `NFTokenMint` (Phase 7), and more are
already on the roadmap. A shared interface means `autofill` - and
future signing/submission logic - works generically across every
future transaction type without repeating the same field-filling
logic per type.

## `XrplPayment` and `XrplTrustSet`

Both are plain, immutable data classes: required fields per the
official specification, optional fields default to `null`, `toJson()`
omits any `null` field rather than sending it explicitly, and
`copyWith()` returns a new instance with specific fields replaced
(used internally by `autofill`).

**Scope decision:** `XrplPayment.amountDrops` is XRP-only for this
sub-version - always a plain string of drops, never an issued-currency
amount object. Payments in other currencies involve XRPL's
cross-currency path-finding, which belongs to Phase 5 (DEX &
Cross-Currency), not here.

`XrplTrustSet.limitValue` is, by contrast, inherently about an issued
currency (that's the entire purpose of a trust line) - `currency`,
`issuer`, and `limitValue` together build the required `LimitAmount`
object.

## Decision Taken: `XrplFeeStrategy`, Not a Single Hardcoded Fee Choice

The official `fee` command reports four already-calculated values at
once (`base_fee`, `minimum_fee`, `open_ledger_fee`, `median_fee`).
Rather than picking one and hardcoding it, `XrplFeeStrategy` exposes
all four as an enum, so a caller who cares about minimizing cost
(accepting the risk of queuing) can choose `minimum`, while the
default (`openLedger`) favors prompt inclusion, per the official
"Reliable Transaction Submission" guidance. Adding all four cost
nothing extra in complexity - the values are already computed by the
server, this enum only selects which field of the response to read.

## Decision Taken: `LastLedgerSequence` = Current Ledger + 4

Confirmed directly from xrpl.org's "Reliable Transaction Submission"
page: automated processes should use a value at least 4 greater than
the last validated ledger index. `autofill` gets both pieces of
information it needs for this - the fee, and the current ledger
index - from a **single call** to the `fee` command, which
conveniently reports `ledger_current_index` alongside the fee data.

## `autofill`: Reuses Existing Queries, Skips What's Already Set

```dart
Future<T> autofill<T extends XrplTransaction>(
  XrplConnection connection,
  T transaction, {
  XrplFeeStrategy feeStrategy = XrplFeeStrategy.openLedger,
})
```

Two existing pieces of this SDK are reused directly, not
reimplemented: `accountInfo` (Phase 3) for the account's current
`Sequence`, and the new `fee` for `Fee`/`LastLedgerSequence`. Any
field already set on the input transaction is left untouched - a
transaction with all three fields already provided never touches the
network at all, verified with a dedicated unit test.

The generic function returns `T`, matching the type it was given
(`XrplPayment` in, `XrplPayment` out). This relies on every concrete
`XrplTransaction` subtype overriding `copyWith` to return its own
type rather than the broader interface type - a pattern Dart allows
for overriding methods, documented directly in the `autofill`
implementation's comments since it isn't obvious from the interface
alone.

## New `fee()` Query

Added to `xrpl_queries.dart` alongside `serverInfo`/`accountInfo`.
Unlike those two, `fee()` returns the **entire** `result` object
rather than a single nested field - every field in it
(`drops`, `levels`, `ledger_current_index`) is potentially useful,
not just one sub-object.

## Test Vectors / Verification

There's no fixed cryptographic value to verify here - correctness
means "does this build the right JSON shape, and does autofill fill
in real, current values." Verification splits accordingly:

- `toJson()`/`copyWith()` for both transaction types: pure unit
  tests, no network, confirming required-only vs. all-fields-set
  output shapes
- `autofill`'s network-skip behavior: a unit test confirming zero
  network calls occur when every field is already provided
- `autofill`'s real fee/ledger lookup: an integration test against
  the public Testnet server, with `sequence` provided explicitly to
  avoid the same funded-account limitation already tracked for
  `accountInfo` in Phase 3 (`accountInfo`'s untested success path)

13 tests total. See
[`test/transactions/models/xrpl_payment_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/transactions/models/xrpl_payment_test.dart),
[`test/transactions/models/xrpl_trust_set_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/transactions/models/xrpl_trust_set_test.dart),
[`test/transactions/xrpl_fee_strategy_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/transactions/xrpl_fee_strategy_test.dart),
[`test/transactions/xrpl_autofill_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/transactions/xrpl_autofill_test.dart),
[`test/src/connection/xrpl_fee_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_fee_integration_test.dart),
and
[`test/src/transactions/xrpl_autofill_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/transactions/xrpl_autofill_integration_test.dart).

## Official Sources

- [xrpl.org - Transaction Common Fields](https://xrpl.org/docs/references/protocol/transactions/common-fields)
- [xrpl.org - Payment](https://xrpl.org/docs/references/protocol/transactions/types/payment)
- [xrpl.org - TrustSet](https://xrpl.org/docs/references/protocol/transactions/types/trustset)
- [xrpl.org - fee method](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/server-info-methods/fee)
- [xrpl.org - Reliable Transaction Submission](https://xrpl.org/docs/concepts/transactions/reliable-transaction-submission)
- `xrpl.js` official documentation (as the client-library convention `autofill` follows)

## Related

- [JSON-RPC Requests & Account/Server Queries](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-3/requests-and-queries/README.md) - `accountInfo`, reused here for `Sequence`
