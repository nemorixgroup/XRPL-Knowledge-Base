# Real-Time Subscription Streams

**Phase:** 3 - Connection Layer  
**Status:** ✅ Done (`0.2.3-dev`)  
**Implementation:** [`lib/src/connection/xrpl_connection.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_connection.dart), [`lib/src/connection/xrpl_subscriptions.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_subscriptions.dart)

## What This Is

`0.2.2-dev` gave the SDK request/response capability - ask a
question, get one answer. Some information isn't naturally a
question-and-answer, though: "tell me every time a new ledger
closes" doesn't have a single answer, it has an ongoing stream of
them. XRPL's `subscribe` mechanism lets a client ask the server to
push messages whenever something happens, instead of polling with
repeated requests.

## The Core Discovery: Two Genuinely Different Kinds of Message

Confirmed directly against the official `subscribe` reference: the
response to the `subscribe` command itself arrives exactly like any
other request's response (matched by `id`, standard envelope). The
actual stream events that follow are a different shape entirely -
they never carry an `id` at all, and are identified instead by a
`type` field:

```
Response to "subscribe" (has "id", matched by XrplConnection.request):
{ "id": 1, "status": "success", "type": "response", "result": {} }

A later, unsolicited event (no "id", identified by "type"):
{ "type": "ledgerClosed", "ledger_index": 7125358, "ledger_hash": "...", ... }
```


`XrplConnection`'s existing `_handleIncomingMessage` (from
`0.2.2-dev`) silently discarded any message without a matching
numeric `id`. This sub-version changes that: a message with an `id`
still resolves a pending request, but a message without one is now
checked for a `type` and routed to the matching event stream instead
of being dropped.

## Stream-to-Event-Type Mapping

| Stream (what you subscribe to) | Event `type` (what arrives) | Exposed as |
|-----------------------------------|------------------------------|-------------|
| `ledger` | `ledgerClosed` | `XrplConnection.ledgerEvents` |
| `transactions` | `transaction` | `XrplConnection.transactionEvents` |
| `transactions_proposed` | `transaction` | `XrplConnection.transactionEvents` (same stream - both share the same event type) |
| `validations` | `validationReceived` | `XrplConnection.validationEvents` |
| `server` | `serverStatus` | `XrplConnection.serverEvents` |

## Decision Taken: Typed Streams per Event Type, Not One Generic Stream

Two designs were weighed before writing any code:

1. **One generic `events` stream**, where the caller checks
   `event['type']` themselves.
2. **Separate, typed streams** (`ledgerEvents`, `transactionEvents`,
   and so on), each already filtered to one event type.

Option 2 was chosen. A generic stream pushes a typo-prone
responsibility onto every caller - comparing `event['type']` against
a string like `'ledgerClosed'` by hand, with no compiler help if it's
misspelled, and no error raised if a case is simply forgotten (the
event is just silently never handled). Typed streams eliminate that
entire class of mistake, and match how official client libraries in
other languages already solve this (for example, Go's `xrpl-go`
client exposes separate stream channels per event type, not one
generic channel).

## Decision Taken: Stream Controllers Created Once, Not per `connect()`

The four `StreamController`s backing `ledgerEvents`,
`transactionEvents`, `validationEvents`, and `serverEvents` are
created in the class's field initializers, not inside `connect()`.
This means a caller's `.listen()` subscription, set up once, keeps
receiving events correctly even across a `disconnect()` followed by a
new `connect()` - the streams themselves are stable, only what feeds
them (the underlying socket) comes and goes.

## What's Intentionally Ignored (and Why)

Four recognized event types have no dedicated stream yet, documented
directly in `_routeEvent`'s doc comment along with their official
source, rather than silently dropped with no explanation anywhere:

- `consensusPhase` (`consensus` stream) - relevant to lower-level
  consensus monitoring, not planned for this SDK yet
- `bookChanges` (`book_changes` stream) - relevant to Phase 5 (DEX &
  Cross-Currency), not this sub-version
- `peerStatusChange` (`peer_status` stream) - admin-only, about
  connected peer `xrpld` servers, not applicable to a client SDK
- `manifestReceived` (`manifests` stream) - validator key rotation
  info, not applicable to a typical client SDK use case

Receiving one of these isn't treated as an error - it's a
feature not built yet, not a failure.

## A Note on Subscriptions and Reconnection

XRPL subscriptions are tied to the specific WebSocket connection they
were made on - the server does not remember them across a
disconnect/reconnect. Since `XrplConnection` does not reconnect
automatically (a design choice already made in `0.2.1-dev`), this
isn't a new problem to solve here: if the connection drops, any
active subscriptions are gone with it, and the caller needs to
re-subscribe (via `subscribeToLedger` and friends) after reconnecting.
Documented directly in `connect()`'s doc comment so it isn't a
surprise later.

## A Test Bug Caught Along the Way

An initial unit test asserted `connection.ledgerEvents` returns the
*same object* on repeated access. This failed - and rightly so: for a
`StreamController.broadcast()` in Dart, each access to `.stream`
returns a new wrapper object, all connected to the same underlying
controller. Object identity was never the right thing to test here.
The test was replaced with one that verifies the actual functional
guarantee a broadcast stream provides: multiple simultaneous
listeners can subscribe without error, all backed by the same
controller underneath.

## Test Vectors / Verification

Unlike prior sub-versions, verification here means "does a real event
actually arrive on the right stream," not a fixed value:

- `ledgerEvents` is verified against a **real, live `ledgerClosed`
  event** from the public Testnet server - the test subscribes, then
  awaits the first event with a 15-second timeout (generous relative
  to XRPL's ~3-5 second ledger close time)
- `serverEvents`, `transactionEvents` (with and without
  `transactions_proposed`), and `validationEvents` are verified for
  successful subscribe/unsubscribe without error; a real event isn't
  awaited for these, since their timing is far less predictable than
  ledger closes and would make the test slow or flaky

8 tests total. See
[`test/connection/xrpl_connection_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/connection/xrpl_connection_test.dart),
[`test/src/connection/xrpl_connection_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_connection_integration_test.dart),
and
[`test/src/connection/xrpl_subscriptions_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_subscriptions_integration_test.dart).

## Official Sources

- [xrpl.org - subscribe](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/subscription-methods/subscribe)
- [xrpl.org - unsubscribe](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/subscription-methods/unsubscribe)
- [xrpl.org - Response Formatting](https://xrpl.org/docs/references/http-websocket-apis/api-conventions/response-formatting)

## Related

- [JSON-RPC Requests & Account/Server Queries](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-3/requests-and-queries/README.md) - the request()/response pattern this contrasts with
- [Connection Lifecycle](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-3/connection-lifecycle/README.md) - why subscriptions don't survive a reconnect
