# AMMCreate & amm_info

**Status:** ✅ Done (0.4.3-dev)  
**Shipped in:** `xrpl_flutter_sdk` [`0.4.3-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#043-dev)  
**Files:**

- [`lib/src/transactions/models/xrpl_amm_create.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/models/xrpl_amm_create.dart)
- [`lib/src/transactions/values/xrpl_asset.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/values/xrpl_asset.dart)
- [`lib/src/transactions/xrpl_owner_reserve_cost_transaction.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_owner_reserve_cost_transaction.dart)
- [`lib/src/transactions/xrpl_autofill.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_autofill.dart)
- [`lib/src/connection/xrpl_amm_info.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_amm_info.dart)
- [`lib/src/connection/xrpl_queries.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_queries.dart)
- [`lib/src/transactions/binary/xrpl_transaction_serializer.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_transaction_serializer.dart)
- [`lib/src/transactions/binary/xrpl_field_definitions.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_field_definitions.dart)

## Summary

`0.4.3-dev` is the third sub-version of Phase 5 (DEX & Cross-Currency) and the first of the AMM ones. It adds `AMMCreate`, which creates an Automated Market Maker liquidity pool for a pair of assets, and `amm_info`, the query that reads a pool back. It also changed how `autofill` computes the fee for this kind of transaction, because `AMMCreate` does not pay the base fee: it must destroy at least the incremental owner reserve.

## Design Decision: the AMMCreate fee

The official documentation requires `AMMCreate` to destroy at least the owner reserve (`reserve_inc`) instead of the normal transaction cost. A normal fee is rejected by the server, so `autofill` needed a different source for this transaction.

Three options were considered:

1. A fixed constant (200000 drops). Simple, but wrong the day the network changes its reserve.
2. A multiple of the fee-levels figures from the `fee` command. It was designed first, then rejected after checking xrpl.js and rippled's own behavior: both use the owner reserve, not a fee-level scaling.
3. Read `validated_ledger.reserve_inc` from `server_state` and use it as the `Fee` (chosen). It follows the network and matches xrpl.js.

The user keeps control: an explicit `fee` is always respected and is not validated. When the fee is null, `autofill` reads the reserve. Fee strategy and load scaling are not applied to these transactions. `reserve_inc` was confirmed against the real Testnet to be an integer number of drops, and the integration test asserts that the filled `Fee` equals it.

A limitation worth stating: the rippled source code itself was not read; the behavior was confirmed through the official documentation, xrpl.js and a discussion in the xrpl4j project, and then against the real network.

## Design Decision: a marker interface

`autofill` has to know which transactions need the owner-reserve fee without changing the `XrplTransaction` interface that every existing model implements. `XrplOwnerReserveCostTransaction` is an empty `abstract interface class`; `XrplAmmCreate` implements it, and `autofill` checks `transaction is XrplOwnerReserveCostTransaction`. Later transactions with the same cost only need to implement the marker. A `serverState` helper was added to `xrpl_queries.dart` to support it.

## Design Decision: XrplAmmCreate fields and validation

`AMMCreate` has the fields `Amount`, `Amount2` and `TradingFee`. An early draft of this work assumed `Asset`/`Asset2`; reading the official documentation corrected it (those names belong to `amm_info` and to other AMM transactions).

`TradingFee` is in units of 1/100000 and ranges from 0 to 1000 (0% to 1%). `XrplAmmCreate` validates at construction, throwing `ArgumentError`, only what can be decided without ledger state:

- The trading fee must be between 0 and 1000.
- Both amounts must be positive.
- At most one of the two assets can be XRP.
- The two assets cannot be the same currency and issuer.

Everything that depends on ledger state is deliberately left to the server and surfaces as its result code: `tecDUPLICATE` (the pool exists), `tecNO_LINE` and `tecUNFUNDED_AMM` (the creator does not hold enough of the token), `terNO_RIPPLE` (the issuer lacks Default Ripple), `tecFROZEN`, `tecNO_AUTH` and others.

A known gap: `XrplCurrencyAmount` has no equality and does no validation of its own, so `XrplAmmCreate` does the comparisons and checks itself.

## Design Decision: XrplAsset

`amm_info` identifies a pool by two assets, written as `{"currency": "XRP"}` or `{"currency": "TST", "issuer": "r..."}`. `XrplAsset` models that: `XrplAsset.xrp()` and `XrplAsset.issued(currency:, issuer:)`. It rejects an empty currency or issuer, and `'XRP'` as an issued currency. It is a separate type from `XrplCurrencyAmount` because an asset has no quantity.

## Design Decision: ammInfo

`ammInfo` takes either `ammAccount` or the `asset` + `asset2` pair; passing a single asset, or neither form, throws `ArgumentError`, as does a `ledgerIndex` that is neither an integer nor a string. `ledgerIndex` stays `Object?`, as in the documentation, and is validated at runtime. The response is parsed into `XrplAmmInfo` (account, both amounts, LP token, trading fee, optional frozen flags, auction slot and vote slots, and the ledger metadata), with `XrplAmmAuctionSlot` and `XrplAmmVoteSlot`. A response that does not have the expected shape raises `XrplConnectionException`, and server errors such as `actNotFound` surface the same way.

The server does not guarantee which of the two assets is `amount` and which is `amount2`, so neither the parser nor the tests assume an order.

## Design Decision: binary codec

`AMMCreate` needed the transaction type code 35, `Amount2` (Amount, nth 11, Field ID `0x6B`) and `TradingFee` (UInt16, nth 5, Field ID `0x15`). The serializer's `Amount` field, previously XRP-only, now accepts either an XRP string or an issued-currency map, which `AMMCreate` requires. Field definitions are never typed by hand: they are generated and checked by `scripts/regenerate_field_definitions.dart` against a live `server_definitions` response, which prints each value with a `[matches known-good value]` marker. The same values were also compared with xrpl-py and xrpl.js, and the expected binary vectors in the tests were produced with xrpl-py, so they come from an independent implementation.

## Design Decision: minimal AccountSet support

A real pool needs the token issuer to have Default Ripple enabled, which is an `AccountSet` with `SetFlag` 8. The SDK has no `AccountSet` model; the complete one belongs to Phase 8 (Account Security). Rather than publish a partial public model that Phase 8 would have to redesign, only the binary support was added: transaction type 3 and the `SetFlag` field (UInt32, nth 33), verified against xrpl-py with a byte-for-byte vector. The integration test and the example each define a small private class for it.

## How the integration tests exercise a real pool

Unlike the offer and path finding tests, a self-issued token is not enough here. The pool creator must hold the token, and an issuer cannot hold its own token, so the test uses two wallets:

1. The issuer enables Default Ripple (`AccountSet`, `SetFlag` 8).
2. The holder opens a trust line (`XrplTrustSet`).
3. The issuer posts an offer selling 100 TST for 10 XRP and the holder takes it (`XrplOfferCreate` twice). `XrplPayment` is XRP-only in this SDK, so this is how the holder receives the token.
4. The holder creates the pool with 5 XRP and 50 TST at a 0.5% trading fee.

## Observed on Testnet

These values come from running the example against the real public Testnet server, not from the documentation:

- The `AMMCreate` fee paid was 200000 drops, which is the network's `reserve_inc` at the time, instead of the roughly 10 drops of a normal transaction.
- The LP token currency is a 40-character hexadecimal code, one per pool, not a three-letter code.
- The LP token balance of the first deposit was 15811.388..., which equals the square root of 5 000 000 drops times 50 TST: the pool computes with the smallest units, not with whole XRP.
- `amm_info` returned the same pool whether it was looked up by asset pair or by the AMM account.

## What 0.4.3-dev Tests Actually Check

1. `XrplAsset`: construction rules, `toJson`, equality.
2. `XrplAmmCreate`: every client-side rule (fee range, positive amounts, one XRP side at most, same-token rejection), `copyWith`, `toJson`, and a serializer vector.
3. Serializer: three `AMMCreate` vectors (XRP plus issued with fee 500, issued plus issued with fee 1000, fee 0) and the minimal `AccountSet` vector, all byte-for-byte against xrpl-py, plus type-error cases.
4. `serverState` against the real Testnet.
5. `autofill` for `AMMCreate` against the real Testnet: the fee equals `reserve_inc`, an explicit fee is kept, and a normal `Payment` still gets a fee below the reserve.
6. `ammInfo`: argument validation, and parsing of the official documentation example response.
7. Real Testnet end to end: AccountSet, TrustSet, two offers, `AMMCreate` (`tesSUCCESS`, an `AMM` node created in the metadata), `ammInfo` by asset pair and by AMM account with the expected amounts and trading fee, a second `AMMCreate` rejected with `tecDUPLICATE`, and `ammInfo` of a missing pool failing.

A flaky unrelated test was also fixed in this sub-version: the `ripplePathFind` integration test failed intermittently with `actNotFound` right after funding, probably because `ripple_path_find` reads the open ledger of a server that can lag the validated ledger the faucet used. The cause is not confirmed; the test now retries a few times.

## Out of Scope for 0.4.3-dev

- A public `AccountSet` model: Phase 8.
- `AMMDeposit` (`0.4.4-dev`), `AMMWithdraw` (`0.4.5-dev`), `AMMVote` (`0.4.6-dev`), `AMMBid` (`0.4.7-dev`).
- Cross-currency `Payment` using the pool: `XrplPayment` remains XRP-only.
- Validation of an explicit `fee` against the reserve, and load scaling of the owner-reserve fee.
- Verifying the parsing of an active auction slot or vote slots against a live pool: the parsers are covered by the official documentation example only.
- Phase 5 closing audit: `0.5.0-dev`.

## Related

- [Path Finding (0.4.2-dev)](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/path-finding/README.md)
- [OfferCreate & OfferCancel (0.4.1-dev)](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/offer-create-offer-cancel/README.md)
- [CHANGELOG entry: `0.4.3-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#043-dev)

## Sources

- [XRPL AMMCreate](https://xrpl.org/docs/references/protocol/transactions/types/ammcreate)
- [XRPL amm_info](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/path-and-order-book-methods/amm_info)
- [XRPL Automated Market Makers](https://xrpl.org/docs/concepts/tokens/decentralized-exchange/automated-market-makers)
- [XRPL server_state](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/server-info-methods/server_state)
- [XRPL AccountSet](https://xrpl.org/docs/references/protocol/transactions/types/accountset)
