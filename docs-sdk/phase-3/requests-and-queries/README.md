# JSON-RPC Requests & Account/Server Queries

**Phase:** 3 - Connection Layer Extended  
**Status:** ✅ Done (`0.2.2-dev`)  
**Implementation:** [`lib/src/connection/xrpl_connection.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_connection.dart), [`lib/src/connection/xrpl_queries.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_queries.dart)

## What This Is

`0.2.1-dev` gave the SDK a live connection that can open and close,
but nothing that could actually talk to the server. This sub-version
teaches it the protocol: send a request, get back the matching
response. `account_info` and `server_info` are the first two real
commands built on top of that, giving the SDK its first genuine
XRPL data; not just an open socket.

## The Official Request/Response Format

```dart
Request: { "id": <client-chosen>, "command": "<name>", ...params }
Response (success): { "id": <same>, "status": "success", "type": "response", "result": {...} }
Response (error): { "id": <same>, "status": "error", "type": "response", "error": "<code>" }
```

Confirmed against the official WebSocket API reference on xrpl.org.

## Decision Taken: One Generic Method, Small Command-Specific Helpers

```dart
Future<Map<String, dynamic>> XrplConnection.request(String command, [Map<String, dynamic> params]);

Future<Map<String, dynamic>> serverInfo(XrplConnection connection, {bool counters});
Future<Map<String, dynamic>> accountInfo(XrplConnection connection, String account, {...});
```

`request()` knows the shared envelope (`id`, `command`, matching
responses back to their request) and nothing else; it has no idea
what `"account_info"` means or where the useful data lives in its
response. `serverInfo` and `accountInfo` are small, focused functions
built on top of it, each knowing just one command's specific shape.
This means every future command (Phase 4's `submit`, Phase 5's
`book_offers`, and so on) reuses the same `request()` plumbing rather
than reimplementing send/match/receive from scratch.

## Matching Responses to Requests by `id`

A single connection can have multiple requests in flight at once
(confirmed with a concurrent-requests test against the real Testnet
server). Each call to `request()` gets its own incrementing `id`, and
a `Map<int, Completer>` tracks which call is waiting for which
response. A single shared stream listener (set up once in `connect()`,
not once per request) reads every incoming message, decodes its `id`,
and resolves the matching `Completer` - unmatched or malformed
messages are safely ignored rather than crashing anything in flight.

## Failure Handling

Three distinct failure modes, each handled explicitly rather than
left to crash or hang silently:

- **Not connected**: `request()` throws immediately if `_channel` is
  null, before attempting to send anything
- **Timeout**: a `defaultRequestTimeout` of 20 seconds (a practical
  choice, not a value xrpl.org specifies) bounds how long a caller
  waits for a response that may never come
- **Connection drops or closes mid-request**: any requests still
  pending at that point are failed explicitly with
  `XrplConnectionException`, rather than left as `Future`s that never
  resolve

## Decision Taken: Unwrap the Response Envelope

Both `serverInfo` and `accountInfo` return only the useful inner data
(`result.info`, `result.account_data`), not the full response
envelope. A caller never needs to know or repeat XRPL's
`result.<field>` wrapping - `print(info['server_state'])`, not
`print(response['result']['info']['server_state'])`. This is a
deliberate, consistent pattern meant to carry forward to every future
query helper.

## Decision Taken: Optional Parameters Default to `null`, Not Sensible Defaults

`accountInfo` has four optional parameters (`ledgerHash`,
`ledgerIndex`, `queue`, `signerLists`) beyond the required `account`.
Each is only added to the outgoing request if the caller actually
provides it, so the simplest call sends the same minimal request the
official documentation's simplest example shows, not a request
padded with invented defaults nobody asked for.

## Test Vectors / Verification

Unlike prior sub-versions, correctness here isn't a fixed cryptographic
value - it's "does this actually talk to a real server correctly."
Verification leans on this SDK's Phase 3 integration test pattern
(`test/src/`, real Testnet calls, kept apart from fast unit tests):

- `server_info` is called for real and checked for expected fields
  (`server_state`, `build_version`, `complete_ledgers`)
- Multiple concurrent `request()` calls are confirmed to each resolve
  with their own correct response
- `account_info` for a freshly generated, never-funded `XrplWallet` is
  confirmed to fail with the official `actNotFound` error - chosen
  specifically because it's stable over time, unlike a hardcoded
  "known funded account" would be, since Testnet resets periodically

**Known gap, tracked intentionally:** `accountInfo`'s success case (a
funded account's real data) is not yet tested, since doing so needs a
way to actually fund a Testnet account first (for example, the
Testnet Faucet/Friendbot). Marked with a `TODO` in the test file
rather than worked around with an unreliable fixture.

15 tests total across `request()`, `serverInfo`, and `accountInfo`
combined. See
[`test/connection/xrpl_connection_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/connection/xrpl_connection_test.dart)
and
[`test/src/connection/xrpl_connection_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_connection_integration_test.dart)
and
[`test/src/connection/xrpl_queries_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_queries_integration_test.dart).

## Official Sources

- [xrpl.org - account_info](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/account-methods/account_info)
- [xrpl.org - server_info](https://xrpl.org/docs/references/http-websocket-apis/public-api-methods/server-info-methods/server_info)
- [xrpl.org - Request Formatting](https://xrpl.org/docs/references/http-websocket-apis/api-conventions/request-formatting)
- [xrpl.org - Response Formatting](https://xrpl.org/docs/references/http-websocket-apis/api-conventions/response-formatting)

## Related

- [Connection Lifecycle](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-3/connection-lifecycle/README.md) - the connect/disconnect foundation this builds on
