# Transaction Submission & Confirmation

**Phase:** 4 - Core Transactions  
**Status:** ✅ Done (`0.3.3-dev`)  
**Implementation:** [`lib/src/connection/xrpl_queries.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_queries.dart) (`tx`, `submit`), [`lib/src/transactions/xrpl_transaction_hash.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_transaction_hash.dart), [`lib/src/transactions/xrpl_submit_and_wait.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_submit_and_wait.dart), [`lib/src/transactions/xrpl_send_transaction.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_send_transaction.dart), [`lib/src/transactions/xrpl_send_payment.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/xrpl_send_payment.dart), [`lib/src/wallet/xrpl_fund_test_wallet.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/wallet/xrpl_fund_test_wallet.dart)

## What this is

`0.3.2-dev` produced a fully signed, network-ready transaction. This
sub-version closes the loop: send it, and know for certain whether it
was accepted. It is also, by a wide margin, this SDK's most
extensively debugged sub-version - four real, independently-confirmed
bugs were found and fixed here, three of them only discoverable by
actually sending real transactions to a real network, not by any
amount of offline verification.

## The Building Blocks

- **`transactionHash(signedTransactionJson)`**: computes a signed
  transaction's identifying hash, using the `0x54584E00` ("TXN\0")
  prefix - a genuinely different prefix from the `0x53545800`
  single-signing prefix used when computing the hash that gets
  *signed*. Confirmed against two independent sources (the official
  hash-prefix table, and a maintained third-party Rust
  implementation's constants) before being trusted.
- **`tx(connection, transactionHash)`** / **`submit(connection, txBlob, {failHard})`**:
  the two official commands. `submit` reports only a *preliminary*
  result; `tx` looks up a transaction's current, possibly-final
  status by hash.
- **`submitAndWait`**: not an official command - a convenience
  matching `xrpl.js`'s function of the same name, built on `submit`
  and `tx`.
- **`sendTransaction`** / **`sendPayment`**: a further convenience
  layer on top of the full pipeline, added specifically because
  chaining `autofill` -> `sign` -> `submitAndWait` by hand, correctly,
  every time, is more than most callers should need to think about.
- **`fundTestWallet`**: funds a wallet via the official Testnet/Devnet
  Faucet, closing a gap every prior sub-version's tests had been
  working around (no funded account to test success paths against).

## Bug 1: Ed25519 Transactions Were Being Signed Incorrectly

**The most serious bug found in this SDK to date.** Every
Ed25519-signed transaction produced by `0.3.2-dev` would have been
rejected by the real network.

### The symptom

A `submit()` integration test using a freshly generated, unfunded
Ed25519 wallet consistently failed with `invalidTransaction` /
`"Invalid signature"` - on two independently-operated Testnet servers
(classic `rippled` and `Clio`). `secp256k1`, using the exact same code
path, worked without issue.

### The investigation

Every hypothesis testable from within the SDK was ruled out, in
order, each with independent verification:

1. The unsigned binary serialization matched an independent Python
   reconstruction, byte for byte
2. The signing hash (`SHA-512Half` of the prefixed serialization)
   matched Python, byte for byte
3. The private-key-to-public-key relationship matched an independent
   re-derivation
4. The signature itself matched byte-for-byte what Python's `pynacl`
   computes for the same private key and the same (incorrectly
   pre-hashed) message
5. A second, independently-operated server (`Clio`) gave the identical
   rejection with a more specific error: `"fails local checks: Invalid signature."`
6. Neither the destination server, nor whether the wallet was freshly
   generated or restored from an already-verified seed, nor whether
   the sending account was funded, changed the outcome

At this point, every internally-checkable thing had been checked and
was self-consistent - which was itself the clue that something in the
*process itself*, not any individual value, was wrong.

### The resolution

`xrpl.js` was installed and used **entirely offline** (signing needs
no network access) to sign the identical transaction with the
identical seed. Its `ripple-binary-codec` package exposes
`encodeForSigning(tx)` directly - the exact bytes it hashes before
signing. Compared byte-for-byte against this SDK's own reconstruction:
**identical**. Yet `xrpl.js`'s own signature did not verify against
`SHA-512Half` of those bytes either.

Testing the remaining hypothesis directly - does `xrpl.js` sign the
*raw* prefixed, serialized bytes, with no separate pre-hashing step -
confirmed it immediately: `xrpl.js`'s signature verified successfully
against the raw bytes.

**The root cause:** `secp256k1` (ECDSA) can only sign a fixed-size
digest, so pre-hashing with `SHA-512Half` is required. `Ed25519`
(EdDSA) is designed to sign messages of *arbitrary length* directly -
it performs its own internal `SHA-512` hashing as part of the
algorithm itself. XRPL does not pre-hash before Ed25519 signing; doing
so anyway produces a signature that is internally self-consistent
(which is why every self-check passed) but signs the wrong message
entirely from the network's point of view.

### The fix

`sign()` in `xrpl_signer.dart` now branches by algorithm: `secp256k1`
still pre-hashes (`XrplHash.sha512Half`) before signing; `Ed25519`
signs the prefixed, serialized bytes directly. `XrplEd25519.sign`'s
doc comment (and its parameter name, `messageHash` -> `message`) was
corrected to stop asserting the now-known-incorrect claim that XRPL
always pre-hashes.

Confirmed fixed with a real, successful Ed25519 transaction on the
public Testnet (`tesSUCCESS`), and the test vector this bug was first
caught by was recomputed with the corrected process and re-verified
byte-for-byte against `pynacl`.

## Bug 2: `autofill`'s Default `LastLedgerSequence` Margin Was Too Tight

The official "Reliable Transaction Submission" page states `4` as the
*minimum* automated processes should use - not a recommended default.
A real transaction, built via the example code's more detailed
pipeline (which performs a couple of extra round-trips beyond the
bare minimum: `accountInfo`, then `fee`), expired using exactly that
minimum.

Research into what real libraries actually default to confirmed the
bare minimum is uncommonly tight in practice: `ripple-lib`
(`xrpl.js`'s predecessor) defaulted to `+8`; its own migration guide
shows a `+75` example for extra safety margin.

**Fix:** `autofill` gained a new `ledgerOffset` parameter, defaulting
to `20` (roughly 60-100 seconds at XRPL's typical ~3-5 second ledger
close time). The official minimum of `4` remains available by passing
it explicitly.

## Bug 3: `submitAndWait` Compared Expiry Against the Wrong Ledger

Even with the increased margin, a transaction still expired - by
exactly one ledger, every time.

**The root cause:** `submitAndWait`'s expiry check used `fee()`'s
`ledger_current_index`. Official documentation confirms this is *the
ledger currently being built* - always at least one ledger ahead of
the latest **validated** one. The official expiry rule ("the XRP
Ledger never includes a transaction in a ledger version whose ledger
index is higher than `LastLedgerSequence`") is defined relative to
validated ledgers specifically - comparing against the in-progress one
declares a transaction expired one ledger before it should be.

**Fix:** the expiry check now uses `serverInfo()`'s
`validated_ledger.seq` (already available from Phase 3's
`serverInfo`, no new query needed) instead of `fee()`'s
`ledger_current_index`.

*(An additional one-time "grace period" retry was added alongside
this fix, as a defense against a suspected server-side indexing lag
between a ledger validating and `tx()` reflecting its contents. It
was later removed after finding no evidence it was ever the actual
fix for any observed failure - the real remaining cause was Bug 4,
below. Documented here as a reminder not to keep unproven defensive
complexity once its actual necessity can't be demonstrated.)*

## Bug 4: `fundTestWallet` Returned Before Funding Actually Validated

Even after Bugs 2 and 3 were both fixed, the example code's "simple"
path - `fundTestWallet` immediately followed by `sendPayment` - still
failed intermittently, by exactly one ledger. The same code using an
already-existing, long-funded wallet instead of a freshly-funded one
worked every time, isolating the cause precisely.

**The root cause:** the Faucet's HTTP response only confirms the
*funding request* was accepted - not that the funding transaction has
actually validated on the ledger yet. Sending a transaction
immediately afterward chains two network-timing-dependent operations
back to back with no verified handoff between them.

**Fix:** `fundTestWallet`'s signature changed from taking a bare
`XrplEndpoint` to taking an `XrplConnection` (it needs one anyway to
confirm funding). After the Faucet's HTTP response, it now actively
polls `accountInfo` - specifically with `ledgerIndex: 'validated'`,
not the default in-progress ledger (the same current-vs-validated
distinction as Bug 3, caught here via the same symptom pattern) -
until the account is confirmed to exist, before returning.

## A Design Decision: `sendTransaction` / `sendPayment`

Once the full pipeline (`autofill` -> `sign` -> `submitAndWait`)
existed and worked, a natural question followed: is asking every
caller to chain three functions correctly, every time, the right
default experience? `sendTransaction<T extends XrplTransaction>`
collapses that into one call for any supported transaction type;
`sendPayment` goes further, building the `XrplPayment` internally so
a caller sending simple XRP never needs to know `XrplPayment` exists.
Both remain optional - the individual steps stay available for
callers who need to inspect or adjust the transaction in between (for
example, checking the autofilled `Fee` before signing), and the
example code deliberately shows both approaches side by side.

`sendPayment`'s parameters are named `senderWallet` and
`destinationAddress`, not a bare `wallet` and `destination` - raised
directly by Stark during review. A full wallet (holds a private key,
needed to sign) and a plain address string (the recipient's public
address only - sending funds never requires access to the
recipient's keys) are different enough concepts to deserve
differently-named parameters, not just different types that happen
to look similar at a glance.

## Test Vectors / Verification

Unlike prior sub-versions, the strongest verification here isn't a
fixed value - it's **real, successful, independently-confirmed
transactions on the public Testnet**, several of them, across
multiple bug-fix iterations. Each fix in this document was confirmed
not just by reasoning, but by watching a real transaction that had
failed under the old code succeed under the new one.

See
[`test/src/connection/xrpl_tx_submit_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_tx_submit_integration_test.dart),
[`test/src/transactions/xrpl_submit_and_wait_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/transactions/xrpl_submit_and_wait_integration_test.dart),
[`test/src/transactions/xrpl_send_payment_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/transactions/xrpl_send_payment_integration_test.dart),
and
[`test/src/wallet/xrpl_fund_test_wallet_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/wallet/xrpl_fund_test_wallet_integration_test.dart).

## Official Sources

- [xrpl.org - submit](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/transaction-methods/submit)
- [xrpl.org - tx](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/transaction-methods/tx)
- [xrpl.org - Reliable Transaction Submission](https://xrpl.org/docs/concepts/transactions/reliable-transaction-submission)
- [xrpl.org - Ledger Header](https://xrpl.org/docs/references/protocol/ledger-data/ledger-header) (current vs. validated ledgers)
- [xrpl.org - Transaction Results](https://xrpl.org/docs/references/protocol/transactions/transaction-results) (`tes`/`tec`/`tem`/`tel` prefixes)
- [xrpl.org - XRP Faucets](https://xrpl.org/docs/tools/xrp-faucets)
- `xrpl.js` (`ripple-binary-codec`'s `encodeForSigning`) - used directly, offline, to isolate Bug 1

## Related

- [Transaction Signing](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/signing/README.md) - produces the input `submit`/`submitAndWait` consume
- [Transaction Model & Autofill](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/transaction-model/README.md) - where `ledgerOffset`'s default lives
