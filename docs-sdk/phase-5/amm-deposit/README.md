# AMMDeposit

**Status:** ✅ Done (0.4.4-dev)  
**Shipped in:** `xrpl_flutter_sdk` [`0.4.4-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#044-dev)  
**Files:**

- [`lib/src/transactions/models/xrpl_amm_deposit.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/models/xrpl_amm_deposit.dart)
- [`lib/src/transactions/binary/xrpl_amount_serializer.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_amount_serializer.dart)
- [`lib/src/transactions/binary/xrpl_transaction_serializer.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_transaction_serializer.dart)
- [`lib/src/transactions/binary/xrpl_field_definitions.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/binary/xrpl_field_definitions.dart)
- [`scripts/regenerate_field_definitions.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/scripts/regenerate_field_definitions.dart)
- [`test/support/xrpl_amm_test_pool.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/support/xrpl_amm_test_pool.dart)

## Summary

`0.4.4-dev` is the fourth sub-version of Phase 5 (DEX & Cross-Currency) and the second AMM one. It adds `AMMDeposit`, the transaction that adds liquidity to an existing pool and gives the depositor LP tokens in return. `AMMDeposit` is the first transaction in this SDK whose valid shape depends on a mode: the official documentation allows exactly six combinations of fields, and any other combination is invalid. Most of the work of this sub-version was therefore about making the invalid combinations impossible to build, and about the new `Issue` binary type that the pool's two assets are written with.

## Design Decision: one named constructor per mode

`AMMDeposit` takes one mode flag, and each mode needs a specific set of fields:

| Mode | Flag | Fields |
|------|------|--------|
| `lpToken` | `tfLPToken` (`0x00010000`) | `LPTokenOut` |
| `singleAsset` | `tfSingleAsset` (`0x00080000`) | `Amount` |
| `twoAsset` | `tfTwoAsset` (`0x00100000`) | `Amount`, `Amount2` |
| `oneAssetLpToken` | `tfOneAssetLPToken` (`0x00200000`) | `Amount`, `LPTokenOut` |
| `limitLpToken` | `tfLimitLPToken` (`0x00400000`) | `Amount`, `EPrice` |
| `twoAssetIfEmpty` | `tfTwoAssetIfEmpty` (`0x00800000`) | `Amount`, `Amount2` (empty pool only) |

Two designs were possible. A single constructor with every field optional and a `flags` argument is flexible, but it lets a caller build a transaction that the server rejects as malformed, and the mistake only shows up after a network round trip. One named constructor per mode (`XrplAmmDeposit.singleAsset(...)`, `XrplAmmDeposit.twoAsset(...)` and so on) was chosen instead: each constructor takes exactly the fields of its mode and sets exactly one flag, so a wrong mix does not compile. This is the same reasoning already applied to `XrplCurrencyAmount` (a dedicated type instead of optional XRP and issued fields) and to the SDK's explicit signing algorithm.

The `tf*` constants are still public, and `flags` is a public field, so the value that is serialized can be inspected.

## Design Decision: AMMDeposit pays the normal fee

`AMMCreate` has to destroy at least the owner reserve, and `0.4.3-dev` added a marker interface so `autofill` reads `reserve_inc` for it. `AMMDeposit` is different: the official documentation example uses a plain `Fee` of 10 drops, and the official tutorial uses ordinary autofill. `XrplAmmDeposit` therefore does not implement `XrplOwnerReserveCostTransaction`, and it goes through the normal fee strategy.

This was a hypothesis until it was checked against the real network, and the first integration test asserts it: the filled `Fee` is lower than the network's `reserve_inc`. The example confirmed the actual cost (see "Observed on Testnet").

## Design Decision: the Issue binary type

The two assets of the pool are serialized as `Asset` and `Asset2`, of type `Issue` (type code 24). An `Issue` identifies an asset without an amount, and it has no length prefix:

- XRP is 20 zero bytes.
- A token is its 20-byte currency code followed by its 20-byte issuer.

`XrplAmountSerializer.encodeIssue` writes it, rejecting an XRP asset that carries an issuer and a token that has none. The serializer reads each of these fields from a map with `currency` and an optional `issuer`, the same shape `XrplAsset.toJson` already produces for `amm_info`.

LP tokens need one more detail: their currency is a 40-character hexadecimal code, not a three-letter one. `encodeCurrencyCode` became public and now checks that a 40-character code is valid hexadecimal, throwing `XrplCryptoException` otherwise, instead of encoding garbage.

Field IDs were never typed by hand. `scripts/regenerate_field_definitions.dart` queries a live `server_definitions` response and prints each value with a `[matches known-good value]` marker, and the same values were compared with xrpl-py:

| Field | Type | Field ID |
|-------|------|----------|
| `Asset` | Issue | `0x03 0x18` |
| `Asset2` | Issue | `0x04 0x18` |
| `LPTokenOut` | Amount | `0x60 0x19` |
| `EPrice` | Amount | `0x60 0x1B` |

`AMMDeposit` itself is transaction type 36. Because canonical order sorts by `(typeCode, fieldCode)`, the `Issue` fields, with type code 24, serialize after every other field of the transaction. This is checked with a test that places `Asset` after `Account`.

`LPTokenOut` accepts only an issued amount (LP tokens are never XRP), and `EPrice` accepts either an XRP or an issued amount, since the price is expressed in the deposit asset.

## Design Decision: what is validated client-side

As in `AMMCreate`, `XrplAmmDeposit` validates at construction, throwing `ArgumentError`, only what can be decided without ledger state:

- The two pool assets must be different.
- Every amount must be positive (compared as `BigInt` for drops and as a number for issued amounts).
- `Amount` and `Amount2` must each be one of the two pool assets.
- In a pair, `Amount` and `Amount2` must be different assets.
- `LPTokenOut` must be an issued amount and positive.
- `EPrice` must be positive.

Everything that depends on the pool or on the depositor's balances is left to the server and surfaces as its result code: `tecAMM_EMPTY`, `tecAMM_NOT_EMPTY`, `tecAMM_FAILED`, `tecUNFUNDED_AMM`, `tecFROZEN`, `tecLOCKED`, `tecNO_AUTH`, `tecINSUF_RESERVE_LINE`, `terNO_AMM`, `temBAD_AMM_TOKENS` and the rest listed in the official documentation. A deposit also creates an LP token trust line for the depositor, which is the reason `tecINSUF_RESERVE_LINE` exists.

## Design Decision: TradingFee is left out

`AMMDeposit` has an optional `TradingFee` field (UInt16). The documentation page does not state which modes it applies to, and the xrpl.js validation source, which would settle it, could not be read while preparing this sub-version. Shipping a field whose valid use is guessed would put the same kind of unverified rule into the public API that the six constructors were designed to avoid. It is left out of `0.4.4-dev` and noted for the Phase 5 closing audit.

## Design Decision: a shared Testnet helper

Every AMM transaction from here on needs a real pool, and a real pool needs the setup that `0.4.3-dev` built inside one test file: two wallets, Default Ripple on the issuer, a trust line, a token transfer through two offers, and the pool creation. That setup moved to `test/support/xrpl_amm_test_pool.dart` (not a test itself, since its name does not end in `_test`), and the `AMMCreate` integration test now uses it too. `AMMWithdraw`, `AMMVote` and `AMMBid` will reuse it without copying the setup again.

The helper checks `tesSUCCESS` at every setup step, so a failure in the preparation is reported there and not as a confusing failure of the transaction under test.

## How the integration tests exercise a real pool

One real pool is created once and every mode runs against it. Each test reads the pool with `ammInfo` before and after its deposit, so the tests do not depend on their order:

1. `singleAsset`: 1 XRP goes in, the token side does not change, LP tokens grow, and the fee is below `reserve_inc`.
2. `twoAsset`: both sides and the LP tokens grow.
3. `lpToken`: the LP supply grows by exactly the requested amount.
4. `oneAssetLpToken`: the LP supply grows by the requested amount, and the XRP taken never exceeds the maximum the caller allowed.
5. `limitLpToken`: a deposit under a generous price cap succeeds and the pool grows.

## Observed on Testnet

These values come from running the example against the real public Testnet server, not from the documentation. The pool started at 5 XRP and 50 TST with 15811.388... LP tokens:

- `singleAsset` of 1 XRP paid a fee of 10 drops, against 200000 for `AMMCreate` in `0.4.3-dev`. The pool went to 6 XRP and 50 TST, and the LP supply to about 17320.131, so that deposit minted about 1508.7 LP tokens.
- `twoAsset` was asked for at most 0.5 XRP and 5 TST. The pool held 6 XRP and 50 TST, so the proportion that fits 0.5 XRP is about 4.1667 TST, and that is what was taken: the pool went to 6.5 XRP and about 54.1667 TST. The amounts of this mode are an upper bound, as the documentation says. It also minted exactly the share the deposit represents (0.5 of 6 XRP, about 8.33% of the supply), because two-asset deposits pay no pool fee.
- `lpToken` of exactly 10 LP tokens moved the supply from about 18763.475 to 18773.475 and took 3465 drops and about 0.0289 TST, which is the same 0.053% share of both sides.

## What 0.4.4-dev Tests Actually Check

1. Serializer: byte-for-byte `AMMDeposit` vectors generated with xrpl-py for the six modes, plus one vector with two issued assets, and a test that `Asset` serializes after `Account`.
2. Serializer errors: `Asset` that is not a map, XRP with an issuer, a token without an issuer, an issuer that is not a string, `LPTokenOut` as a string, a non-hexadecimal 40-character currency, and a wrongly typed `EPrice`.
3. `XrplAmmDeposit`: the flags of each constructor, `toJson` for each mode, optional fields left out of the JSON, and `copyWith`.
4. `XrplAmmDeposit` validation: same asset on both sides, zero and negative amounts, an amount that is not in the pool, a second amount that is not in the pool, the same asset in both amounts, XRP as `lpTokenOut`, a zero `lpTokenOut`, a zero `ePrice`, and a valid pool of two issued assets.
5. Two checks that the model's JSON serializes to the expected bytes (`singleAsset` and `lpToken`).
6. Real Testnet: one deposit for each of the five modes described above.

In total, 42 new tests (405 to 447).

## Known intermittent failure

While merging this sub-version, the CI run failed once in the `0.4.1-dev` offer integration test with `tefPAST_SEQ` (the sequence number had already passed) and passed on a re-run. `autofill` reads the sequence with `accountInfo` from the open ledger of the connected server, and the public Testnet is a cluster, so a server that lags a moment can return the previous sequence right after a transaction. This is the second intermittent failure of this kind (`ripplePathFind` was the first) and it is not caused by this sub-version. It is recorded for the Phase 5 closing audit (`0.5.0-dev`), where the SDK's behavior in this situation will be decided.

## Out of Scope for 0.4.4-dev

- `tfTwoAssetIfEmpty` on a real pool: it only works on an empty pool, which needs `AMMWithdraw` (`0.4.5-dev`) to produce. It is covered by the codec and model tests only.
- `TradingFee`: see the design decision above.
- Checking that `ePrice` is expressed in the deposit asset: it is only checked to be positive.
- Observing the server's error codes for a rejected deposit on a real pool.
- `AMMWithdraw` (`0.4.5-dev`), `AMMVote` (`0.4.6-dev`), `AMMBid` (`0.4.7-dev`).
- Consolidating the positive-amount check that `XrplAmmCreate` and `XrplAmmDeposit` both implement, and giving `XrplCurrencyAmount` equality and validation of its own: Phase 5 closing audit (`0.5.0-dev`).

## Related

- [AMMCreate & amm_info (0.4.3-dev)](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-create/README.md)
- [CHANGELOG entry: `0.4.4-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#044-dev)

## Sources

- [XRPL AMMDeposit](https://xrpl.org/docs/references/protocol/transactions/types/ammdeposit)
- [XRPL amm_info](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/path-and-order-book-methods/amm_info)
- [XRPL Automated Market Makers](https://xrpl.org/docs/concepts/tokens/decentralized-exchange/automated-market-makers)
- [XRPL Binary Format](https://xrpl.org/docs/references/protocol/binary-format)
- [xrpl-py](https://github.com/XRPLF/xrpl-py)
