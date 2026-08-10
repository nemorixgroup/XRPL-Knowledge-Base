# X-Address Derivation

**Phase:** 2 - Addresses  
**Status:** ✅ Done (`0.1.2-dev`)  
**Implementation:** [`lib/src/address/xrpl_x_address.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/address/xrpl_x_address.dart), [`lib/src/address/xrpl_network.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/address/xrpl_network.dart)

## What This Is

A classic address ("r...") only identifies an *account*. Many
real-world destinations (exchanges, payment processors) share one
account across many users, and distinguish them with a separate
"destination tag" number that has to be supplied alongside the
address on every transaction. If a sender forgets that tag, funds can
arrive at the right account but become unattributable to the intended
recipient - a common, costly mistake in practice.

X-addresses solve this by encoding the account, an optional tag, and
which network (Mainnet or Testnet) the address is meant for into a
single base58 string, so there's nothing separate left to forget or
mix up.

## The Algorithm

```
network prefix (2 bytes: Mainnet 0x05 0x44, or Testnet 0x04 0x93)

Account ID (20 bytes, same derivation as Classic Address)
flag (1 byte: 0x00 = no tag, 0x01 = 32-bit tag present)
tag (4 bytes, little-endian; zero bytes if no tag)
reserved (4 zero bytes, for a possible future 64-bit tag)
= 31 bytes total
v XrplBase58.encodeWithChecksum
X-Address ("X..." Mainnet, "T..." Testnet)
```

## Decision Taken

```dart
static String deriveFrom(
  Uint8List publicKey, {
  required XrplNetwork network,
  int? tag,
})
```

Reuses [`XrplClassicAddress.accountIdFromPublicKey`](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/classic-address/README.md#decision-taken)
for the Account ID (an X-address encodes the same account, just with
more context) and [`XrplBase58.encodeWithChecksum`](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/base58-codec/README.md#why-raw-and-checksummed-versions-are-separate)
for the final encoding - the same Base58Check-style scheme already
used for seeds and classic addresses, differing only in the type
prefix and payload contents.

## A Discovery: the Payload Is 31 Bytes, Not 30

Before implementing, the assumed layout was `network prefix (2) +
Account ID (20) + flag (1) + tag (4) + reserved (4) = 30 bytes`. The
official `ripple-address-codec` source (the pull request that
originally added X-address support) shows the reserved field is
**4 bytes**, not the initially assumed padding that would have made
the total 30 - the correct total, confirmed by matching an official
worked example exactly, is **31 bytes**. This was caught during
research, before any code was written, by reading the actual
reference implementation rather than working from a summarized
assumption.

## Why the Tag Is Little-Endian

Every other multi-byte value in this SDK so far (the 4-byte sequence
counters in secp256k1 key derivation, for example) is big-endian. The
destination tag is the one exception: XRPL's X-address specification
(XLS-5d, "Standard for Tagged Addresses") defines it explicitly as
little-endian. This is called out directly in the code and tests so
it isn't mistaken for an inconsistency or a bug later.

## Why `tag` Is Optional, Not Required

Unlike the signing algorithm parameter used everywhere else in this
SDK (always required, never inferred, to avoid the "same seed,
different address" ambiguity), `tag` is an optional (`int?`)
parameter here. Omitting a tag is a legitimate, common state - not
every destination uses tags - so treating it as required would force
callers to pass a meaningless placeholder value instead of expressing
"no tag" directly.

## Input Validation

`deriveFrom` validates that `tag`, when provided, is within the
32-bit unsigned range (`0` to `4294967295`, `0xFFFFFFFF`). A negative
tag or one exceeding this range throws `XrplCryptoException` rather
than silently truncating or wrapping the value - consistent with this
SDK's rule of never accepting malformed input and producing output
that merely *looks* valid. Public key length validation is inherited
directly from `XrplClassicAddress.accountIdFromPublicKey`.

## A Test-Vector Bug Caught by an Unexpected Failure

The first version of the test suite reused the public key from the
Classic Address worked example (`ED9434...FA32`, corresponding to
`rDTXLQ7ZKZVKz33zJbHjgVShjsBnqMBhmN`) as if it were the public key
behind the official X-address worked example
(`rGWrZyQqhTp9Xu7G5Pkayo7bXjH4k4QYpf`) - two different accounts from
two different official documentation sources, mixed together by
mistake.

The test suite caught this immediately: `pre_commit.ps1` reported an
assertion failure with the actual output differing from the expected
value starting at the third character. Rather than adjusting the
expected value to match, the discrepancy was investigated first - the
actual output was independently recomputed in Python and matched
exactly, confirming `XrplXAddress` itself was correct and the error
was entirely in how the test fixture was assembled. The test file was
then restructured to clearly separate:

- Vectors built directly from an official classic address's
  independently-decoded Account ID (true official vectors, verified
  byte-for-byte against the published examples)
- A vector chained from the Classic Address test's public key,
  labeled explicitly as independently computed (via the same Python
  script), not as an official published example

This is a concrete example of why this SDK treats an unexpected test
failure as a signal to investigate the *data*, not just the code -
the bug here was in test construction, not implementation.

## Test Vectors / Verification

| Network | Source | Tag |
|---------|--------|-----|
| Mainnet | Official `ripple-address-codec` X-address PR | Maximum (`4294967295`) |
| Testnet | Official `ripple-address-codec` X-address PR | Small (`123`) |
| Mainnet | Chained from the official Classic Address vector; independently computed | Maximum (`4294967295`) |

11 tests total. See
[`test/address/xrpl_x_address_test.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/test/address/xrpl_x_address_test.dart).

## Official Sources

- [`ripple-address-codec` PR #14](https://github.com/ripple/ripple-address-codec/pull/14/files) - the original implementation and worked examples this code matches exactly
- [XLS-5d: Standard for Tagged Addresses](https://github.com/XRPLF/XRPL-Standards/issues/6) - the proposed standard defining the byte layout and little-endian tag encoding
- [xrpl.org - Addresses](https://xrpl.org/docs/concepts/accounts/addresses)

## Related

- [Classic Address Derivation](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-2/classic-address/README.md) - the Account ID this builds on
- [Base58 Codec](https://github.com/nemorixgroup/XRPL-Knowledge-Base/blob/main/docs-sdk/phase-1/base58-codec/README.md) - the checksummed encoding reused here
