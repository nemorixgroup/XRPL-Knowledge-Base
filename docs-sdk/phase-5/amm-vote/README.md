# AMMVote (0.4.6-dev)

**Status:** ✅ Done  
**Shipped in:** 0.4.6-dev (Phase 5, DEX & Cross-Currency)  
**Files:**
- `lib/src/transactions/models/xrpl_amm_vote.dart`
- `lib/src/transactions/binary/xrpl_field_definitions.dart`
- `lib/src/transactions/binary/xrpl_transaction_serializer.dart`
- `scripts/regenerate_field_definitions.dart`
- `test/transactions/models/xrpl_amm_vote_test.dart`
- `test/transactions/binary/xrpl_transaction_serializer_test.dart`
- `test/src/transactions/models/xrpl_amm_vote_integration_test.dart`
- `example/phase5/amm_example.dart`

## Summary

0.4.6-dev adds `XrplAmmVote`, the transaction that votes on the trading
fee of an AMM pool (transaction type 38). Liquidity providers use it to
propose a fee for the pool, and the pool's fee becomes the average of
the latest votes weighted by how many LP tokens each voter holds. It is
the smallest AMM transaction of Phase 5: it has no modes and no flags,
and it needed no new binary fields.

Test count went from 489 to 512 (23 new tests: 5 codec vectors, 14
model tests and 4 Testnet integration tests).

## What the transaction does

Fields, per the official page:

| Field | Type | Required | Description |
|---|---|---|---|
| `Asset` | Issue | Yes | One of the assets in the pool |
| `Asset2` | Issue | Yes | The other asset in the pool |
| `TradingFee` | UInt16 | Yes | The proposed fee, in units of 1/100,000; the maximum is 1000 (1%) |

Voting rules, per the official AMM documentation:

- Only liquidity providers can vote, and a vote weighs as much as the
  share of the pool's LP tokens the voter holds.
- Up to eight votes are counted: the ones from the accounts holding the
  most LP tokens.
- The pool's fee is recalculated as the weighted average of the latest
  votes every time someone places a vote, and only then: LP token
  balances can change without the fee being updated.
- The allowed range is 0% to 1%, in increments of 0.001%.

Result codes the page lists: `tecAMM_EMPTY`, `tecAMM_INVALID_TOKENS` (the
sender holds none of the pool's LP tokens), `tecAMM_FAILED_VOTE` (eight
votes already exist from accounts holding more LP tokens than the
sender), `temBAD_FEE` and `terNO_AMM`.

## Design decisions

### 1. A single constructor

AMMVote has no modes and no flags, so `XrplAmmVote` is one plain
constructor, unlike the seven named constructors of `XrplAmmWithdraw`.
There is no invalid mix of fields to prevent: the three fields are
always required.

### 2. No `Flags` is sent

The official page documents no flags for this transaction. Its example
contains `"Flags": 2147483648` without explaining it, and the xrpl-py
vector used by the codec tests carries no `Flags` field. The model
therefore sends none, the same as `XrplAmmCreate`.

### 3. The codec needed only the transaction type

`Asset`, `Asset2` and `TradingFee` were already defined by earlier AMM
transactions. The only addition is the transaction type code, 38, which
was never typed by hand: `scripts/regenerate_field_definitions.dart`
reads it from the live `server_definitions`, and prints
`AMMVote = 38 [matches known-good value]` against the value the official
xrpl-py definitions give.

### 4. `TradingFee` is validated from 0 to 1000

The model rejects a fee outside 0 to 1000 with an `ArgumentError`. The
official page states the maximum but not a minimum; the official xrpl.js
validation accepts 0 to its maximum constant, xrpl-rust accepts 0 to
1000, and the XRPL concept page gives the range as 0% to 1%. A fee of 0
is therefore valid, and Testnet accepted it.

`XrplAmmVote.maxTradingFee` is not a second copy of the number: it
points at `XrplAmmCreate.maxTradingFee`, since both transactions share
the same limit.

### 5. Ledger-dependent rules are left to the server

The model also rejects a pool whose two assets are the same. Everything
that depends on the ledger (no such pool, the sender holding no LP
tokens, the eight-vote limit) is not checked locally and surfaces as the
server's result code.

### 6. The transaction cost is the normal one

The page states no special cost, unlike AMMCreate, so `autofill` needs no
special handling. Testnet confirmed it: the vote paid 10 drops, against
200000 for AMMCreate. The integration test checks that the autofilled
fee is lower than the network's `reserve_inc`.

### 7. The pool's vote slots are not exposed yet

`amm_info` reports the pool's vote slots, but `XrplAmmInfo` does not
expose them. Adding them is on the 0.5.0-dev audit list; meanwhile the
tests read the pool's resulting `tradingFee`, which is what a vote is
for.

## How the integration tests exercise a real pool

The tests reuse the shared helper `test/support/xrpl_amm_test_pool.dart`:
two funded wallets (an issuer with Default Ripple enabled and a holder
with a TST trust line), the holder creates the pool with 5 XRP and 50
TST at a 500 (0.5%) trading fee, and is its only liquidity provider.

The first two tests share one pool: since the holder owns every LP token,
each of its votes fully decides the pool's fee, so the tests do not
depend on their order. The weighted-average test builds a pool of its own
because it adds a second provider. The last test only needs a funded
account that holds no LP tokens.

## Observed on Testnet

- **Fee:** 10 drops for a vote, a normal transaction cost.
- **A sole provider decides the fee:** the pool was created at 500, and
  after the holder voted 300 `amm_info` reported 300. The limits 0 and
  1000 were both accepted.
- **The fee is a weighted average:** in the example, the holder voted 300
  and a second provider, who had deposited 1 XRP and held about 8.7% of
  the LP tokens (the holder kept about 91.3%), voted 1000. The pool's
  fee became 361, which matches 0.913 x 300 + 0.087 x 1000 (about 361).
- **A second provider moves the fee by its share:** in the integration
  test, where the holder's vote is still 500, a second provider holding
  about 9% voting 1000 moved the fee above 500 but well below 750. The
  test checks only that range, because the exact value depends on the
  server's rounding.
- **An account without LP tokens cannot vote:** the transaction was
  validated with `tecAMM_INVALID_TOKENS`. `sendTransaction` returned that
  result instead of throwing.

## What the tests check

Codec (5 tests in the serializer test file):
- Vectors from xrpl-py for fees 500, 0 and 1000, and for a pool of two
  issued assets.
- A `TradingFee` given as a string is rejected.

Model (14 tests):
- `toJson`: the pool assets and fee, no `Flags`, a fee of 0 sent and not
  omitted, and `Sequence`, `Fee` and `LastLedgerSequence` only when set.
- `copyWith`, including that the original is not mutated.
- Validation: `maxTradingFee` is 1000, the same asset twice is rejected,
  fees of -1 and 1001 are rejected, the limits 0 and 1000 are accepted,
  and a pool of two issued assets is accepted.
- Serialization through the model, for fees 500 and 0, against the
  xrpl-py vectors.

Testnet integration (4 tests):
- A sole provider votes 300, the pool's fee becomes 300, and the vote
  pays the normal fee.
- The limits 0 and 1000 are accepted.
- A second provider moves the fee by its share of the LP tokens.
- An account holding no LP tokens fails with `tecAMM_INVALID_TOKENS`.

## Out of scope

- **`tecAMM_FAILED_VOTE`.** It needs eight accounts holding more LP
  tokens than the voter, which is too much setup for a Testnet run.
- **`vote_slots` in `ammInfo`.** Planned for the 0.5.0-dev audit.
- **MPT assets.** The official page allows an MPT as `Asset`; this SDK
  covers XRP and issued currencies only.
- **AMMBid.** Planned for 0.4.7-dev.

## Related

- [AMMCreate](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-create/README.md)
- [AMMDeposit](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-deposit/README.md)
- [AMMWithdraw](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/amm-withdraw/README.md)

## Sources

- AMMVote transaction: https://xrpl.org/docs/references/protocol/transactions/types/ammvote
- AMM ledger entry (`TradingFee`, `VoteSlots`): https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/amm
- Automated Market Makers concept (archived xrpl-dev-portal source, for the voting rules and the 0% to 1% range): https://git.inftf.org/ARCHIVE/xrpl-dev-portal/raw/commit/3da5fde7a8251756d4137a5d1b6a3e7746c5f55b/content/concepts/tokens/automated-market-makers.md
- xrpl.js AMMVote validation: https://app.unpkg.com/xrpl@5.0.0/files/src/models/transactions/AMMVote.ts
- xrpl-rust AMMVote model: https://docs.rs/crate/xrpl-rust/latest/source/src/models/transactions/amm_vote.rs
- xrpl-py (expected vectors and the transaction type code): https://github.com/XRPLF/xrpl-py
- Live `server_definitions` from the public Testnet server, via `scripts/regenerate_field_definitions.dart`
