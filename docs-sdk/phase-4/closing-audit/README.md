# Phase 4 Closing Audit

**Phase:** 4 - Core Transactions  
**Status:**  ✅ Done (`0.4.0-dev`)  

## What This Is

The same audit pattern used to close Phases 1, 2, and 3: a review
pass over everything built in Phase 4 (`0.3.1-dev` through
`0.3.3-dev`), checking for missing edge cases, error message clarity,
and documentation accuracy, followed by a review of test coverage for
gaps or accidental duplication - plus, this time, a deferred
consolidation task carried over from earlier sub-versions.

## Method

Every file added or changed during Phase 4 was reviewed against the
same three questions used in every prior phase's audit:

1. Missing edge cases - is there any invalid input the code doesn't
   explicitly reject?
2. Clear error messages - does every exception explain what went
   wrong?
3. Documentation accuracy - does every doc comment still correctly
   describe what the code does?

## Result Summary

| File | Outcome |
|------|---------|
| `xrpl_transaction.dart` | 1 change: doc comment updated to mention `sendTransaction`, not just `autofill` |
| `xrpl_payment.dart` | 1 change: documented why `destinationTag` is deliberately unvalidated at construction |
| `xrpl_trust_set.dart` | Audited, no changes needed |
| `xrpl_field_definitions.dart` | 1 change: broken doc link fixed |
| `xrpl_binary_primitives.dart` | 3 real findings: unvalidated `UInt16`/`UInt32` ranges, unvalidated hex format, broken doc link |
| `xrpl_amount_serializer.dart` | 4 real findings, one serious (silent data corruption), plus 2 broken doc links |
| `xrpl_transaction_serializer.dart` | 2 real findings: unchecked type casts, broken doc link |
| `xrpl_signer.dart` | 2 real findings: missing Account-mismatch validation, stale version reference |
| `xrpl_autofill.dart` | 2 real findings: unvalidated `ledgerOffset`, unchecked casts on server response fields |
| `xrpl_submit_and_wait.dart` | 2 real findings: unvalidated `pollInterval`, unchecked casts |
| `xrpl_send_transaction.dart` | Audited, no changes needed |
| `xrpl_send_payment.dart` | 1 change: documented why the Account-mismatch check can never trigger from this entry point |
| `xrpl_fund_test_wallet.dart` | 2 findings: `algorithm` silently ignored when `wallet` is provided (documented), `maxAttempts`/`attemptDelay` made configurable and validated |

Three of thirteen files needed no changes at all. This audit found
proportionally more real findings than any prior phase's - a natural
consequence of Phase 4 introducing this SDK's largest volume of new,
first-time logic (a full binary codec, two signing algorithms, and a
multi-step network pipeline) in a single phase.

## The Most Serious Finding: Silent Data Corruption

`XrplAmountSerializer._encodeCurrencyCode` converted a 3-character
currency code to bytes via `currency.codeUnits` without checking each
unit was a valid ASCII byte (0-127). `Uint8List.fromList` does not
throw for an out-of-range value - it silently truncates to the low 8
bits. A currency code containing any non-ASCII character (an accented
letter, a symbol) would be encoded as a **different, incorrect**
currency code, with no error at any point - the kind of bug that
produces a transaction that succeeds, but does the wrong thing.

Fixed by validating every code unit is within `0x00`-`0x7F` before
encoding, throwing `XrplCryptoException` with the offending value
otherwise.

## A Recurring Category: Unchecked Type Casts on External Data

Across `xrpl_transaction_serializer.dart`, `xrpl_autofill.dart`, and
`xrpl_submit_and_wait.dart`, the same pattern recurred: an `as` cast
on a value either from a hand-buildable map (`XrplTransactionSerializer`)
or a server response (`accountInfo`, `fee`, `server_info`), with no
type check first. Each was replaced with an explicit `is!` check and
a clear `XrplCryptoException`/`XrplConnectionException`, consistent
with the exact same pattern already established and applied during
the Phase 3 audit (`XrplConnection._handleIncomingMessage`) - this
phase's audit found the same category of risk recurring in new code,
confirming it's worth checking for deliberately in every future
phase's audit too, not just assuming it was "fixed once."

## A Design Tension, Resolved Without a Breaking Change

`XrplPayment.destinationTag` (a `UInt32` field) was found unvalidated
at construction. The natural fix - validate in the constructor - was
not available without removing `const` from `XrplPayment`'s
constructor, since Dart `const` constructors cannot contain
conditional validation logic. Removing `const` would break any
existing `const XrplPayment(...)` usage (including this SDK's own
tests and examples), an unacceptable breaking change to a
already-published class.

**Resolution:** validation was added where the value is actually
*used* - `XrplBinaryPrimitives.encodeUInt32`, which now rejects any
out-of-range `UInt32` value with a clear `XrplCryptoException`. This
covers `destinationTag` (and any other unvalidated `UInt32` field)
without ever needing to touch `XrplPayment`'s constructor. Documented
directly in both files, so a future maintainer understands *why*
validation lives one layer away from where the field is defined,
rather than assuming it was simply forgotten.

## A New Validation With Real Teeth: Account-Mismatch in `sign()`

`sign()` previously had no way to detect if a transaction's `Account`
field didn't match the wallet actually being used to sign it - a
mistake that would produce a mathematically valid signature for the
*wrong account*, only surfacing later as a confusing network
rejection (echoing the exact kind of confusing `invalidTransaction`
rejection this SDK spent significant effort diagnosing during the
`0.3.3-dev` Ed25519 investigation, for an unrelated reason).

Adding this check immediately caught a real, pre-existing bug in this
SDK's own test suite: a test signed a transaction whose `Account` was
a hardcoded string from an unrelated official example, not the
address of the wallet actually doing the signing - undetected since
`0.3.2-dev` because nothing previously verified that relationship.
Fixed by using `wallet.classicAddress` directly instead of a separate
hardcoded string, removing the possibility of drift entirely.

`sendPayment` was confirmed, and documented, to never be able to
trigger this check - it always builds `Account` from the same
`senderWallet` passed to `sign()`, so the two can never disagree.

## Deferred Consolidation, Completed: `XrplHexCodec`

Three files (`xrpl_signer.dart`, `xrpl_transaction_hash.dart`,
`xrpl_submit_and_wait.dart`) each contained an identical private
bytes-to-hex implementation, deliberately deferred rather than fixed
in the moment during `0.3.2-dev`/`0.3.3-dev` to avoid scope creep
mid-feature. Consolidated here into `XrplHexCodec`
(`lib/src/codec/`), alongside `XrplBase58` as a general-purpose codec.

**Deliberately not applied to `xrpl_binary_primitives.dart`**: that
file's private hex parser carries additional responsibility (format
validation - odd length, non-hex characters) the shared codec doesn't
have. Consolidating it would have meant either losing that validation
or complicating the shared codec with a responsibility most callers
don't need. The right scope for consolidation is "genuinely identical
duplicates," not "superficially similar code."

## Test Suite Review: One Cross-Layer Propagation Gap, Closed

Following the same check established during the Phase 1 audit
(confirming each layer's errors are independently tested at the
layer above, not just assumed to propagate), `sendTransaction`/
`sendPayment` were found to have only ever been tested against the
shallowest possible failure (a "not connected" `XrplConnectionException`
from `autofill`). A new test confirms `sendTransaction` also
propagates a genuinely different exception type from a deeper layer
- `sign()`'s new `XrplCryptoException` for an Account mismatch -
built to reach that check without any network calls at all, by
providing every field `autofill` would otherwise need to look up.

No accidental duplication was found elsewhere: integration tests for
`sendPayment`, `submitAndWait`, and the manual `submit`/`tx` pipeline
each exercise a genuinely different layer of the same overall
pipeline.

## Status

This closes the full Phase 4 audit. Phase 4 is complete as of
`0.4.0-dev`.

## Related

- [Transaction Model & Autofill](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/transaction-model/README.md)
- [Transaction Signing](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/signing/README.md)
- [Transaction Submission & Confirmation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-4/submission/README.md)
- [Phase 3 Closing Audit](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-3/closing-audit/README.md); where the unchecked-cast pattern was first caught and fixed
