# Connection Lifecycle

**Phase:** 3 - Connection Layer Foundation  
**Status:** ✅ Done (`0.2.1-dev`)  
**Implementation:** [`lib/src/connection/xrpl_endpoint.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_endpoint.dart), [`lib/src/connection/xrpl_connection.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/connection/xrpl_connection.dart), [`lib/src/exceptions/xrpl_connection_exception.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/exceptions/xrpl_connection_exception.dart)

## What This Is

Everything built in Phases 1 and 2 is offline, deterministic math -
generate a seed, derive a key, compute an address, always the same
output for the same input, no notion of "connected" or
"disconnected." Phase 3 is where the SDK starts talking to a real
XRPL server for the first time. This first sub-version covers only
the connection's lifecycle; opening it, closing it, knowing whether
it's open - not sending or receiving any XRPL-specific data yet.

## Why a New `XrplEndpoint`, Not an Extended `XrplNetwork`

`XrplNetwork` (`address/`) already models Mainnet vs. Testnet, but
only because those are the two networks XLS-5d defines a distinct
X-address prefix for. Devnet has no such prefix; stretching
`XrplNetwork` to a third value would leave an undefined question
("what X-address prefix does Devnet use?") baked into a type that
should never need to answer it.

`XrplEndpoint` is a separate, purpose-built enum for "which server do
I connect to," genuinely a three-way choice (Mainnet, Testnet,
Devnet), each carrying its official public WebSocket URL:

| Network | WebSocket URL |
|---------|----------------|
| Mainnet | `wss://xrplcluster.com/` |
| Testnet | `wss://s.altnet.rippletest.net:51233/` |
| Devnet | `wss://s.devnet.rippletest.net:51233/` |

Verified against
[xrpl.org's official public servers page](https://xrpl.org/docs/tutorials/public-servers).

## A New Exception Category: `XrplConnectionException`

Every exception this SDK has thrown through Phase 2 has been
`XrplCryptoException` - and every one of those means "the input or
data itself is wrong" (a bad checksum, an invalid length). A
connection failure is a fundamentally different kind of problem: the
input can be perfectly valid and the request can still fail because a
server is unreachable, slow, or drops mid-session. `XrplConnectionException`
keeps these distinguishable, so calling code can react differently;
retrying usually makes sense for a dropped connection; it never makes
sense for a checksum mismatch.

## Why `package:web_socket_channel`, Not `dart:io`

Same reasoning already applied to entropy generation in Phase 1
(`dart:math`'s `Random.secure` over `dart:io`): `dart:io` does not
exist in a browser context, so any dependency on it blocks Flutter
Web compilation entirely. `package:web_socket_channel`, maintained by
the Dart team, abstracts WebSocket so the same code works unmodified
on mobile, desktop, and (eventually) web.

## Decision Taken

```dart
class XrplConnection {
  XrplConnection(this.endpoint);
  final XrplEndpoint endpoint;

  bool get isConnected;
  Future<void> connect();
  Future<void> disconnect();
}
```

Deliberately minimal for this sub-version: no `send`/`receive` yet.
Two behaviors worth calling out:

- `connect()` throws `XrplConnectionException` if already connected -
  calling it twice without disconnecting first is treated as a
  caller error, not silently ignored or silently reconnected.
- `disconnect()` is a safe no-op if not currently connected - "make
  sure we're disconnected" is a reasonable thing to want regardless
  of current state, so it doesn't need to be an error.

## Why This Sub-Version Stops Here

Sending and receiving JSON-RPC requests (and the SDK's first real
queries, `account_info` and `server_info`) are deliberately deferred
to `0.2.2-dev`, grouped with their first actual use, rather than
shipped in isolation with nothing using them yet; the same reasoning
already applied when merging secp256k1 and Ed25519 into a single
Phase 1 sub-version.

## Test Vectors / Verification

Unlike every prior sub-version, there's no cryptographic value to
verify here. Correctness means "does this actually open and close a
real connection." This SDK's first **integration tests** open a real
WebSocket connection to the public Testnet server, kept in a separate
file from the pure unit tests:

```
test/connection/xrpl_connection_test.dart <- unit tests, no network
test/src/connection/xrpl_connection_integration_test.dart <- real Testnet connection
```

The `test/src/` tree mirrors `lib/src/` specifically for tests that
depend on network availability; a failure there can mean "the public
Testnet server is temporarily down," not necessarily "this SDK's code
is broken," so keeping them separate from the fast, deterministic
unit tests makes that distinction visible at a glance.

13 tests total. See
[`test/connection/xrpl_endpoint_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/connection/xrpl_endpoint_test.dart),
[`test/exceptions/xrpl_connection_exception_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/exceptions/xrpl_connection_exception_test.dart),
[`test/connection/xrpl_connection_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/connection/xrpl_connection_test.dart),
and
[`test/src/connection/xrpl_connection_integration_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/src/connection/xrpl_connection_integration_test.dart).

## Official Sources

- [xrpl.org - Public Servers](https://xrpl.org/docs/tutorials/public-servers), official WebSocket URLs
- [xrpl.org - Get Started Using HTTP / WebSocket APIs](https://xrpl.org/docs/tutorials/get-started/get-started-http-websocket-apis)
- [`package:web_socket_channel`](https://pub.dev/packages/web_socket_channel)

## Related

- [X-Address Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/x-address/README.md), where `XrplNetwork` (the address-specific, two-value type) is defined
