# AMMWithdraw (0.4.5-dev)

**Status:** ✅ Done  
**Shipped in:** 0.4.5-dev (Phase 5, DEX & Cross-Currency)   
**Files:**
- `lib/src/transactions/models/xrpl_amm_withdraw.dart`
- `lib/src/transactions/binary/xrpl_field_definitions.dart`
- `lib/src/transactions/binary/xrpl_transaction_serializer.dart`
- `scripts/regenerate_field_definitions.dart`
- `test/transactions/models/xrpl_amm_withdraw_test.dart`
- `test/transactions/binary/xrpl_transaction_serializer_test.dart`
- `test/src/transactions/models/xrpl_amm_withdraw_integration_test.dart`
- `example/phase5/amm_example.dart`

## Summary

0.4.5-dev adds `XrplAmmWithdraw`, the transaction that removes
liquidity from an AMM pool (transaction type 37). It is the counterpart
of [AMMDeposit](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-deposit/README.md): a liquidity provider gives
LP tokens back to the pool and receives one or both of its assets.

The release also adds the binary codec support the transaction needs:
the transaction type code and one new field, `LPTokenIn`.

Test count went from 447 to 489 (42 new tests: 10 codec vectors, 27
model tests and 5 Testnet integration tests).

## Withdraw modes

AMMWithdraw is controlled by exactly one mode flag, and each flag
allows only one combination of fields. The values were checked against
the official page and against xrpl-py.

| Constructor | Flag | Value | Fields |
|---|---|---|---|
| `lpToken` | tfLPToken | `0x00010000` | `LPTokenIn` |
| `withdrawAll` | tfWithdrawAll | `0x00020000` | none |
| `oneAssetWithdrawAll` | tfOneAssetWithdrawAll | `0x00040000` | `Amount` |
| `singleAsset` | tfSingleAsset | `0x00080000` | `Amount` |
| `twoAsset` | tfTwoAsset | `0x00100000` | `Amount`, `Amount2` |
| `oneAssetLpToken` | tfOneAssetLPToken | `0x00200000` | `Amount`, `LPTokenIn` |
| `limitLpToken` | tfLimitLPToken | `0x00400000` | `Amount`, `EPrice` |

Any other combination makes the server answer `temMALFORMED`.

## Design decisions

### 1. Seven named constructors, one per mode

Same approach as `XrplAmmDeposit`. Each constructor takes exactly the
fields its mode needs, so an invalid mix of fields does not compile,
and the flag is set by the constructor instead of by the caller. The
flag values are exposed as public constants (`tfLpToken`, `tfWithdrawAll`
and so on) for code that works with the raw `Flags`.

### 2. A zero `Amount` is valid in `oneAssetWithdrawAll`

Re-reading the official page showed that, in tfOneAssetWithdrawAll, the
`Amount` may be 0: the transaction then succeeds if it withdraws any
positive amount, and it fails if it cannot pay at least the specified
amount. The first draft validated like AMMDeposit (positive amounts
only) and would have rejected that case. The model now allows zero only
in that mode; every other amount must be positive.

### 3. `LPTokenIn` is an Amount field

`LPTokenIn` has type Amount (6) and field code 26, so its field ID is
`0x60 0x1A`. The value was never typed by hand: it comes from
`scripts/regenerate_field_definitions.dart`, which reads the live
`server_definitions` and prints `[matches known-good value]` against
xrpl-py 5.2.0. The LP token amount is encoded as an issued currency,
with the 40-hex LP currency code. The amount serializer needed no
change, because it already supported hex currency codes from AMMDeposit.

### 4. The fee is the normal transaction cost

The official page states no special cost for AMMWithdraw, unlike
AMMCreate. Testnet confirmed it: the transaction paid 10 drops, against
200000 for AMMCreate. The integration test checks that the autofilled
fee is lower than the network's `reserve_inc`.

### 5. Validation helpers are still duplicated

`_requirePositive` and `_requireAmount` now exist in `XrplAmmCreate`,
`XrplAmmDeposit` and `XrplAmmWithdraw`, and the withdraw version differs
(it allows zero in one mode). Merging them is deferred to the 0.5.0-dev
audit, together with an equality and validation story for
`XrplCurrencyAmount`.

### 6. `EPrice` is only checked to be positive

The model does not check which asset `EPrice` is expressed in. The
official page describes it as the minimum effective price, in LP tokens
returned, to pay per unit of the asset to withdraw, but gives no worked
example. The check is also on the 0.5.0-dev audit list.

## How the integration tests exercise a real pool

The tests reuse the shared helper `test/support/xrpl_amm_test_pool.dart`
from AMMDeposit: two funded wallets (an issuer with Default Ripple
enabled, and a holder with a TST trust line), the issuer sells 100 TST
for 10 XRP, the holder crosses that offer, and the holder creates the
pool with 5 XRP and 50 TST at a 0.5% trading fee. The holder is the only
liquidity provider.

The first three tests share one pool and read it with `ammInfo` before
and after, so they do not depend on their order. The last two build a
pool of their own because they change who provides liquidity or delete
the pool.

## Observed on Testnet

- **Fee:** 10 drops for `lpToken`, a normal transaction cost.
- **`lpToken`:** redeeming 10 LP tokens from a pool of 5 XRP and 50 TST
  (15811.388... LP in circulation) paid 3162 drops of XRP and about
  0.0316 TST, and the LP in circulation dropped by exactly 10.
- **`singleAsset`:** asking for 500000 drops paid 499999. The server
  rounds in the pool's favor, so a withdrawal can pay a drop or two less
  than requested, never more. The token side did not change, and about
  811.6 LP tokens were burned, which is more than the proportional share
  because one-sided withdrawals pay a pool fee.
- **`withdrawAll` by the only provider:** ended in `tesSUCCESS`, the
  metadata contained a `DeletedNode` of type AMM, and `ammInfo` then
  failed because the pool no longer existed.
- **`oneAssetWithdrawAll` with an `Amount` of 0:** succeeded for a second
  provider who had deposited 1 XRP, returning that provider's whole stake
  as XRP.

## What the tests check

Codec (10 tests in the serializer test file):
- One vector per mode, generated with xrpl-py (7).
- `LPTokenIn` field order in the serialized transaction.
- An `LPTokenIn` given as a string is rejected.
- An `LPTokenIn` without an issuer is rejected.

Model (27 tests):
- The flag constants.
- `toJson` for every constructor, including optional fields.
- `copyWith`.
- Validation: the two assets must differ, amounts must belong to the
  pool and be positive, `Amount` and `Amount2` must be different assets,
  `EPrice` must be positive, `LPTokenIn` must be an issued amount.
- Zero `Amount` accepted only in `oneAssetWithdrawAll` (XRP and TST),
  negative amounts rejected, and the pool-asset check still applied.
- Serialization vectors for `singleAsset`, `lpToken` and `withdrawAll`
  through the model.

Testnet integration (5 tests):
- `lpToken` redeems an exact LP amount and pays the normal fee.
- `singleAsset` withdraws XRP, within 10 drops of the request and never
  more, with the token side unchanged.
- `twoAsset` withdraws both assets, using amounts computed from the
  pool's current ratio.
- `oneAssetWithdrawAll` with an `Amount` of 0 returns a second
  provider's stake.
- `withdrawAll` by the only provider deletes the pool.

## Known intermittent failure

While running `pre_commit` for this release, one run failed in the
OfferCancel integration test from 0.4.1 with `tefPAST_SEQ`, and passed
on the next run. The cause is Testnet cluster lag: `autofill` reads the
`Sequence` through `accountInfo` without a ledger index, so a server
that is slightly behind can return a sequence that was already used. It
is not related to AMMWithdraw and it is on the 0.5.0-dev audit list,
together with the intermittent `actNotFound` seen in path finding.

## Out of scope

- **`oneAssetLpToken` and `limitLpToken` on Testnet.** The official page
  does not define how `Amount` is interpreted in tfOneAssetLPToken, nor
  the unit and direction of `EPrice` in tfLimitLPToken, and its only
  example is tfTwoAsset. Both modes are covered by the codec and model
  tests, but not by integration tests, to avoid inventing values.
- **AMMDeposit's tfTwoAssetIfEmpty.** It needs an empty pool, and
  withdrawing everything normally deletes the pool, so an empty pool
  still cannot be produced with two wallets.
- **MPT assets.** The official page allows an MPT as `Asset`; this SDK
  covers XRP and issued currencies only.
- **AMMVote, AMMBid and AMMDelete.** Planned for 0.4.6-dev, 0.4.7-dev
  and later.

## Related

- [AMMCreate](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-create/README.md)
- [AMMDeposit](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-deposit/README.md)

## Sources

- AMMWithdraw transaction: https://xrpl.org/docs/references/protocol/transactions/types/ammwithdraw
- AMMDeposit transaction: https://xrpl.org/docs/references/protocol/transactions/types/ammdeposit
- amm_info method: https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/path-and-order-book-methods/amm_info
- xrpl-py (expected vectors): https://github.com/XRPLF/xrpl-py
- Live `server_definitions` from the public Testnet server, via `scripts/regenerate_field_definitions.dart`
