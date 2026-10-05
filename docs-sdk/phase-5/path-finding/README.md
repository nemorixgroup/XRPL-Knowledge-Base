# Path Finding: ripple_path_find & path_find

**Status:** ✅ Done (0.4.2-dev)  
**Shipped in:** `xrpl_flutter_sdk` [`0.4.2-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#042-dev)  
**Files:**

- [`lib/src/transactions/values/xrpl_path_step.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/values/xrpl_path_step.dart)
- [`lib/src/transactions/values/xrpl_source_currency.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/transactions/values/xrpl_source_currency.dart)
- [`lib/src/connection/xrpl_queries.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_queries.dart)
- [`lib/src/connection/xrpl_path_find.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_path_find.dart)
- [`lib/src/connection/xrpl_path_find_amounts.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_path_find_amounts.dart)
- [`lib/src/connection/xrpl_connection.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_connection.dart)

## Summary

`0.4.2-dev` is the second sub-version of Phase 5 (DEX & Cross-Currency). It adds XRPL's two path-finding commands, which discover how a cross-currency payment could be routed through order books and trust lines: `ripple_path_find` (a single request/response snapshot) and `path_find` (a streaming subscription that keeps sending updates as ledger conditions change). Both are server queries, not transaction types, in the same category as `server_info` or `tx`. Along the way the work fixed a routing problem in `XrplConnection` that would have silently dropped every `path_find` streaming update, and tightened several validations after reviewing the code against the official documentation.

## Design Decision: two commands, two shapes

`ripple_path_find` fits the existing query pattern in `xrpl_queries.dart` directly: one request, one response, the full `result` returned (like `fee`, `tx` and `submit`). `path_find` is a subscription with its own lifecycle (`create`, `status`, `close`), so it lives in its own file, `xrpl_path_find.dart`, as three helpers (`pathFindCreate`, `pathFindStatus`, `pathFindClose`) built on top of the generic `XrplConnection.request`. Follow-up updates arrive on a new typed stream, `XrplConnection.pathFindEvents`, following the same one-stream-per-event-type pattern as `ledgerEvents` and the others.

Per the official documentation, both commands share two caveats that are repeated in the dartdocs: returned paths are not guaranteed to be optimal (an untrusted server could return suboptimal ones, so results can be compared across independent servers), and path finding is unnecessary for XRP-only payments, which transfer directly.

## Design Decision: XrplPathStep and XrplPath

The shared data type both commands return, and that a `Payment` transaction's `Paths` field consumes, has a three-level hierarchy: a `PathStep` is one transition, a `Path` is an array of steps (one complete route), and a `PathSet` is an array of paths (all candidate routes). `XrplPathStep` and `XrplPath` model the first two levels, in `lib/src/transactions/values/` next to `XrplCurrencyAmount`.

`XrplPathStep` holds three optional strings (`account`, `currency`, `issuer`) and validates at construction, throwing `ArgumentError`:

- `account` must not be combined with `currency` or `issuer` in the same step (stated in the official specification).
- `currency: 'XRP'` must not be combined with an `issuer`, since the documentation lists the currency + issuer combination as valid for non-XRP currencies only.

The valid combinations are therefore: `account` alone, `currency` alone, `currency` + `issuer`, and `issuer` alone. A step with no fields set is allowed, since the specification does not forbid it. The legacy `type` and `type_hex` fields (a binary bitmask) are not modeled: the official documentation marks them deprecated, and they carry no information beyond which of the three fields are present.

`XrplPath` is a thin wrapper around `List<XrplPathStep>` rather than a type alias, to give it a documented place in the public API and a natural home for `toJson()`. The `PathSet` level (a list of `XrplPath`) is a plain `List<XrplPath>` for now, accepted by `pathFindCreate`'s `paths` parameter.

## Design Decision: XrplSourceCurrency

Each entry of `ripple_path_find`'s `source_currencies` list has a mandatory `currency` and an optional `issuer`. The list accepts at most 18 entries, a limit enforced in `ripplePathFind` rather than in `XrplSourceCurrency`, since a single entry cannot know how many siblings it has (the same reasoning that keeps this class `const`).

The `currency: 'XRP'` + `issuer` rule from `XrplPathStep` was deliberately not copied here: the official `ripple_path_find` documentation does not state it for `source_currencies`, and this project only encodes rules it can cite.

## Design Decision: ripplePathFind validation

`ripplePathFind` takes `sourceAccount`, `destinationAccount` and `destinationAmount` as required parameters, and `sourceCurrencies`, `sendMax`, `ledgerHash` and `ledgerIndex` as optional ones, sending only the fields actually provided (the same convention as `accountInfo`). Validation runs before any request is sent, each throwing `ArgumentError`:

- `sourceCurrencies` and `sendMax` are mutually exclusive, per the official specification.
- `sourceCurrencies` accepts at most 18 entries.
- `destinationAmount` and `sendMax` must be of a supported type (see the next section).

The `domain` parameter (PermissionedDEX amendment) is out of scope for this sub-version.

## Design Decision: the "-1" shortcut

The official documentation lets `destination_amount` request "a path to deliver as much as possible, while spending no more than the amount specified in `send_max`", with the value `-1`: `"-1"` as the amount itself for XRP, or `-1` as the `value` of an issued-currency amount. Two consequences are encoded in the SDK:

- `destinationAmount` accepts either an `XrplCurrencyAmount` or the literal string `"-1"`. The literal is the XRP form; for an issued currency, the maximum is requested with an `XrplCurrencyAmount.issued` whose `value` is `'-1'`. Both forms are documented in the dartdocs of `ripplePathFind` and `pathFindCreate`.
- `sendMax` accepts only an `XrplCurrencyAmount`. The documentation defines the `-1` shortcut for `destination_amount` only, and `send_max` is the cap on what the sender will spend, so it has to be a real amount. Passing `"-1"` (or anything else) throws an `ArgumentError`.

Both commands apply exactly the same rules, so they live in one internal file, `xrpl_path_find_amounts.dart` (`pathFindDestinationAmountJson` and `pathFindSendMaxJson`, marked `@internal` and not exported from the package barrel), instead of being duplicated.

## Design Decision: routing path_find updates by type, before id

`XrplConnection` routes incoming messages with a simple rule established in Phase 3: a message with an `id` is the response to a request, and a message without one is a subscription event identified by its `type`. `path_find` breaks that rule. Per the official specification, every asynchronous update for an open subscription carries `"type": "path_find"` and also reuses the `id` of the original `create` request. With the `id` check running first, such an update was matched against a request whose response had already been delivered and removed, so it was dropped without any error.

Four approaches were considered:

1. A generic fallback: when an `id` has no pending request, try routing by `type`. It would fix the bug, but implicitly: any future message that reuses an `id` would also fall into that path, with nothing in the code saying why.
2. Inverting the order for all messages (check `type` first). It touches the central logic that already protects `serverInfo`, `accountInfo`, `fee`, `tx`, `submit` and `ripplePathFind`, all tested and verified, for a problem that affects one command.
3. Handling the raw stream separately for `path_find`. It breaks the encapsulation of `XrplConnection` and creates a race between two parsing systems.
4. An explicit check for `type == 'path_find'` before the generic `id` routing, emitting on a dedicated stream.

Option 4 was chosen: it states its intent in the code, leaves every existing path untouched, and also gives `XrplConnection` a place to track whether a subscription is active, which the next decision needs. Normal responses to the `create`, `status` and `close` requests are unaffected, since they arrive as ordinary responses, not with type `path_find`.

## Design Decision: one active path_find request per connection

The official documentation states that only one pathfinding request can be active per connection, and that opening a new one closes the previous one automatically. The server does not return an error in that case, so a second `create` would silently replace the first. `pathFindCreate` makes this explicit: it throws a `StateError` if a subscription is already open, unless `replaceExisting: true` is passed. Invalid arguments are reported first, so an `ArgumentError` is never masked by the state check.

To know whether a subscription is open without teaching the generic `request()` anything about `path_find`, `XrplConnection` exposes `hasActivePathFindSubscription` (read-only) and an `@internal` method, `markPathFindSubscriptionActive`, called by the helpers: `pathFindCreate` marks it open once the server confirms the `create`, and `pathFindClose` marks it closed once the server confirms the `close`. `disconnect()` resets it, since XRPL subscriptions do not survive a reconnect. The flag is maintained by the helpers rather than set when an update arrives, because an update still in flight after a `close` would otherwise flip it back to open.

Known limitations, kept deliberately simple:

- The flag only reflects subscriptions managed through the helpers; a raw `request('path_find', ...)` is not tracked.
- If `pathFindClose` fails (for example with `noPathRequest`), the flag is left unchanged; a caller in that situation can pass `replaceExisting: true` explicitly.
- The `analyzer` rule `use_setters_to_change_properties` flags `markPathFindSubscriptionActive`; a setter would need a same-named getter, and the public getter is named for what it answers, so the lint is silenced with an `ignore` and a comment explaining why.

## Findings during review

Three corrections came from reviewing the code against the official documentation after it already worked and its tests passed:

- `XrplPathStep` accepted `currency: 'XRP'` with an `issuer`, although the documentation lists the currency + issuer combination as non-XRP only. Fixed in the constructor, and `copyWith` inherits the rule because it delegates to the constructor.
- `ripplePathFind` reused one conversion function for `destinationAmount` and `sendMax`, so `sendMax: '-1'` was accepted, although the shortcut is only defined for `destination_amount`. Split into two rules.
- The dartdocs described the literal `"-1"` as the way to request the maximum for any currency. The documentation says it is the XRP form, with `value: '-1'` for issued currencies. Dartdocs corrected, and a test added for the issued-currency form.

## Design Decision: how the integration tests exercise path finding

Like the `OfferCreate` tests in `0.4.1-dev`, the integration tests request a currency issued by the destination account itself (`TST`), avoiding any trust line setup. The consequence is that no real route exists between the two unrelated funded wallets, so the server correctly returns `alternatives: []`. These tests therefore verify the request flow, the subscription lifecycle and the message routing, not the quality of the paths found. The `path_find` tests fund the two accounts once for the whole group, in a `setUpAll`.

`flutter_test`'s `setUpAll` and `group` do not accept a `timeout` parameter (only `test` does), so the longer timeout needed for funding two wallets from the Testnet faucet is set with a library-level `@Timeout` annotation at the top of the file.

One behavior was observed on the real Testnet server rather than taken from the documentation: the server sends streaming updates for an open `path_find` subscription even when `alternatives` is empty. The integration test that waits for an update depends on it. The test confirms that updates reach `pathFindEvents`; it does not assert that the update carries the reused `id`, so that detail of the specification is not independently verified by the test suite.

## What 0.4.2-dev Tests Actually Check

1. `XrplPathStep`: every valid field combination, the account exclusion rule (with `currency`, with `issuer`, with both), the XRP + issuer rule, `copyWith` enforcing both rules, `toJson` omitting unset fields, and equality.
2. `XrplPath`: wraps an ordered list of steps, `toJson` preserves order, and equality is sensitive to step order and count.
3. `ripplePathFind` validation: `sourceCurrencies` and `sendMax` together are rejected, 19 `sourceCurrencies` entries are rejected while exactly 18 are accepted, an invalid `destinationAmount` type is rejected, the literal `"-1"` and an issued-currency amount with `value: '-1'` are accepted for `destinationAmount`, and `sendMax` rejects both an invalid type and the literal `"-1"`.
4. `pathFindCreate` validation: the same amount rules as above, and a list of `paths` is accepted.
5. `pathFindCreate` state handling: a new connection has no active subscription, a `StateError` when one is open, `replaceExisting: true` and a closed state both skip it, and an invalid argument is reported before the `StateError`.
6. `pathFindStatus` and `pathFindClose` throw `XrplConnectionException` without a connection, and a failed `pathFindClose` leaves the subscription flag unchanged.
7. Real Testnet integration, `ripple_path_find`: a response shaped per the specification with empty `alternatives` for two unrelated funded accounts, and `srcActNotFound` for an unfunded source.
8. Real Testnet integration, `path_find`: the full `create` / `status` / `close` lifecycle with the flag following each step, a second `create` throwing `StateError` unless `replaceExisting` is true, `status` and `close` failing when nothing is open, and a real asynchronous update arriving on `pathFindEvents`.

## Out of Scope for 0.4.2-dev

- The `domain` parameter (PermissionedDEX amendment) in both commands.
- A test that finds a real, non-empty path: it needs trust lines and funded order books between several accounts, which the self-issued currency approach deliberately avoids.
- Using `XrplPath` in a cross-currency `Payment`'s `Paths` field: `XrplPayment` remains XRP-only in this sub-version.
- AMM transaction types (`AMMCreate`, `AMMDeposit`, `AMMWithdraw`, `AMMVote`, `AMMBid`): `0.4.3-dev` through `0.4.7-dev`.
- Phase 5 closing audit: `0.5.0-dev`.

## Related

- [OfferCreate & OfferCancel (0.4.1-dev)](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-5/offer-create-offer-cancel/README.md)
- [CHANGELOG entry: `0.4.2-dev`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/CHANGELOG.md#042-dev)

## Sources

- [XRPL ripple_path_find](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/path-and-order-book-methods/ripple_path_find)
- [XRPL path_find](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/path-and-order-book-methods/path_find)
- [XRPL Paths (fungible tokens)](https://xrpl.org/docs/concepts/tokens/fungible-tokens/paths)
- [XRPL Payment](https://xrpl.org/docs/references/protocol/transactions/types/payment)
- [XRPL Specifying Currency Amounts](https://xrpl.org/docs/references/protocol/data-types/basic-data-types#specifying-currency-amounts)
