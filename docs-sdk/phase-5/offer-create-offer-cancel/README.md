# OfferCreate & OfferCancel

**Status:** ✅ Done (0.4.1-dev)  
**Shipped in:** `xrpl_flutter_sdk` [`0.4.1-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#041-dev)  
**Files:**
- [`lib/src/transactions/values/xrpl_currency_amount.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/values/xrpl_currency_amount.dart)
- [`lib/src/transactions/models/xrpl_offer_create.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/models/xrpl_offer_create.dart)
- [`lib/src/transactions/models/xrpl_offer_cancel.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/models/xrpl_offer_cancel.dart)
- [`lib/src/transactions/binary/xrpl_transaction_serializer.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_transaction_serializer.dart)
- [`lib/src/transactions/xrpl_submit_and_wait.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_submit_and_wait.dart)

## Summary

`0.4.1-dev` is the first sub-version of Phase 5 (DEX & Cross-Currency). It adds the SDK's first decentralized-exchange transaction types, `OfferCreate` and `OfferCancel`, verified byte-for-byte against XRPL's official binary specification and against the real public Testnet server. The work also surfaced and fixed a real bug in the SDK's submission pipeline (`submitAndWait`), unrelated to Offers themselves but discovered while investigating a flaky integration test.

## Design Decision: XrplCurrencyAmount as a new shared value type

`TakerGets` and `TakerPays` can each independently be plain XRP or an issued currency, unlike `XrplPayment.amountDrops`, which was deliberately scoped to XRP-only in Phase 4. Representing this with separate optional fields would allow invalid combinations (both set, or neither). Instead, `XrplCurrencyAmount` was introduced as a small closed value type (`XrplCurrencyAmount.xrp(drops)` / `XrplCurrencyAmount.issued(currency, issuer, value)`), living in a new `lib/src/transactions/values/` folder, a sibling to `models/` and `binary/`. The folder is intentionally general-purpose: it is expected to hold `XrplPathStep` (Path Finding, `0.4.2-dev`) and similar shared value types later in Phase 5, following the same "shared value type, not a full transaction" shape.

## Design Decision: XrplOfferCreate / XrplOfferCancel field scope

Both follow the established `XrplTransaction` pattern from Phase 4 (const constructor, `copyWith`, `toJson`, implements `XrplTransaction`).

- `XrplOfferCreate`: `account`, `takerGets`, `takerPays` (all required), plus optional `expiration`, `offerSequence` (to replace an existing Offer in one transaction), `sequence`, `fee`, `lastLedgerSequence`, `flags`.
- `XrplOfferCancel`: `account`, `offerSequence` (both required), plus the standard optional autofill-managed fields.
- `XrplOfferCreateFlags`: `tfPassive` (`0x00010000`), `tfImmediateOrCancel` (`0x00020000`), `tfFillOrKill` (`0x00040000`). `tfImmediateOrCancel` and `tfFillOrKill` are mutually exclusive per the official specification.

A subtlety confirmed against the official specification and worth calling out explicitly: `OfferCancel` always returns `tesSUCCESS`, even when `offerSequence` does not match any existing Offer. Confirming an Offer was actually removed requires inspecting the transaction's metadata for a `DeletedNode` of type `Offer`, not just the top-level result code. This is demonstrated directly in the integration tests (see below).

## Design Decision: Binary serializer extension

The existing canonical-ordering serializer (sort by `(typeCode, fieldCode)` as separate numbers, never by comparing already-encoded Field ID bytes) needed no structural change, only new field/type coverage: `Expiration` and `OfferSequence` as `UInt32`, and a new `_encodeEitherAmount` helper so `TakerGets`/`TakerPays` can each independently serialize as either drops (XRP, 8-byte MSB-set integer) or an issued-currency amount (160-bit currency code + issuer AccountID + 64-bit custom float), reusing the same `Amount` encoding already proven correct for `Payment.Amount` in Phase 4.

This was verified byte-for-byte against XRPL's official `OfferCreate` worked example before being trusted, per this project's standing verification practice: no field definition or encoding rule is implemented from memory or assumption.

**Official worked example (raw transaction hex, 220 bytes):**

```
120007220008000024001abed82a2380bf2c2019001abed764d55920ac9391400
0000000000000000000000000000555344000000000a20b3c85f482532a9578d
bb3950b85ca06594d1654000000037e11d60068400000000000000a732103ee8
3bb432547885c219634a1bc407a9db0474145d69737d09ccdc63e1dee7fe37446
30440220143759437c04f7b61f012563afe90d8dafc46e86035e1d965a9ced282
c97d4ce02204cfd241e86f17e011298fc1a39b63386c74306a5de047e213b0f29
efa4571c2c8114dd76483facdee26e60d8a586bb58d09f27045c46
```


Decoded and cross-checked field by field (`TransactionType=OfferCreate(7)`, `Flags=524288`, `Sequence=1752792`, `Expiration=595640108`, `OfferSequence=1752791`, `TakerPays={USD, rvYAfWj5gh67oV6fW32ZzP3Aw4Eubs59B, 7072.8}`, `TakerGets=15000000000 drops`, `Fee=10`, `SigningPubKey`, `TxnSignature`, `Account=rMBzp8CgpE441cp5PVyA9rpVV7oT8hP3ys`) before constructing the equivalent Dart transaction map and confirming `XrplTransactionSerializer.serialize()` reproduces the exact 220-byte official hex.

`OfferSequence`'s Field ID is a genuinely useful edge case for the canonical-ordering rule: it has a 2-byte Field ID (type code 2, field code 25, both above the 1-nibble threshold), while `Expiration`'s Field ID is 1 byte. Sorting by the encoded bytes directly would place them in the wrong relative order; sorting by `(typeCode, fieldCode)` as separate integers, as already implemented, places them correctly.

## Design Decision: self-issued currency for integration tests

Testing `OfferCreate`/`OfferCancel` against the real Testnet needs an issued currency on at least one side of the offer. An issuer never needs a trust line for its own issued currency, so the integration tests use a self-issued `TST` currency (the test wallet is both account and issuer) rather than funding and configuring a second wallet purely to be a currency issuer. This keeps the integration tests self-contained to a single funded wallet.

## Design Decision: --concurrency=1 in pre_commit.ps1

Kept as a defense against cross-file wallet-sharing risk between integration test files that use the same Testnet wallet (`xrpl_offer_integration_test.dart` and `xrpl_send_payment_integration_test.dart`), even though the actual root cause found this sub-version (see Bug below) turned out to be unrelated to concurrency. `flutter test --coverage` parallelizes across files by default; `--concurrency=1` removes that risk category entirely at a small, acceptable cost to local test run time.

## Bug found and fixed: submitAndWait silently ignored rejected submissions

While debugging a flaky integration test (`OfferCancel` with a deliberately non-existent `OfferSequence`), a genuine SDK bug was found and fixed, independent of the flakiness itself.

**Investigation:** the test failed with a `LastLedgerSequence` expiry, not a framework timeout. Two rounds of race-condition mitigation were tried first (`--concurrency=1` in `pre_commit.ps1`, then a dedicated second Testnet wallet) and neither fixed it, including against a completely fresh, never-before-used wallet. Since the failure was deterministic rather than random, the actual transaction content was re-examined: the test used `offerSequence: 999999999`, which was *larger* than the transaction's own auto-filled `Sequence`. Per the official specification, `OfferSequence` (in `OfferCancel`, or `OfferCreate`'s optional replace-offer field) must be strictly less than the transaction's own `Sequence`, or the server rejects the transaction immediately and definitively as `temBAD_SEQUENCE`.

`submitAndWait` was discarding `submit()`'s return value entirely and always entering the polling loop, so a definitive rejection (`tem*`/`tef*`, per the official `engine_result` classification: never relayed or retried) was silently treated the same as a pending transaction, and only surfaced once `LastLedgerSequence` expired minutes later.

**Fix:** `submitAndWait` now inspects `submit()`'s `engine_result` and fails fast with a clear `XrplConnectionException` whenever it starts with `tem` or `tef`, rather than waiting out the full expiration window. `tes*`, `tec*` (claimed fee, included in ledger despite failure) and `ter*` (local/retriable) results still enter the polling loop, since only `tem*`/`tef*` are guaranteed by the specification to never reach a ledger.

## What 0.4.1-dev Tests Actually Check

1. `XrplOfferCreate`/`XrplOfferCancel` construction, `copyWith`, `toJson`, default/optional field handling.
2. `XrplCurrencyAmount.xrp()`/`.issued()` construction and JSON shape.
3. `XrplTransactionSerializer` produces the exact 220-byte official `OfferCreate` worked-example hex, byte-for-byte.
4. Canonical field ordering for `Expiration` (1-byte Field ID) vs. `OfferSequence` (2-byte Field ID) is correct.
5. `OfferCancel` binary serialization (`OfferSequence` field only, beyond the standard envelope).
6. Real Testnet integration: creating an Offer and confirming a `CreatedNode` of type `Offer` in the transaction metadata, then canceling it and confirming a `DeletedNode` of type `Offer`.
7. Real Testnet integration: `OfferCancel` against a non-existent `OfferSequence` returns `tesSUCCESS` with no `DeletedNode` of type `Offer` present, i.e. the transaction succeeds without actually removing anything.
8. Real Testnet integration: `submitAndWait` throws immediately (well under the `LastLedgerSequence` expiration window) when the server rejects a malformed `OfferCancel` with `temBAD_SEQUENCE`, instead of waiting out the full timeout.

## Out of Scope for 0.4.1-dev

- Path Finding (`ripple_path_find`, `path_find`): `0.4.2-dev`.
- AMM transaction types (`AMMCreate`, `AMMDeposit`, `AMMWithdraw`, `AMMVote`, `AMMBid`): `0.4.3-dev` through `0.4.7-dev`.
- Phase 5 closing audit: `0.5.0-dev`.

## Related

- [CHANGELOG entry: `0.4.1-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#041-dev)

## Sources

- [XRPL OfferCreate](https://xrpl.org/docs/references/protocol/transactions/types/offercreate)
- [XRPL OfferCancel](https://xrpl.org/docs/references/protocol/transactions/types/offercancel)
- [XRPL Binary Format - Field Order](https://xrpl.org/docs/references/protocol/binary-format#field-order)
- [XRPL Amount Fields (Currency Amounts)](https://xrpl.org/docs/references/protocol/binary-format#amount-fields)
- [XRPL Transaction Results (tes/tec/ter/tem/tef)](https://xrpl.org/docs/references/protocol/transactions/transaction-results)
