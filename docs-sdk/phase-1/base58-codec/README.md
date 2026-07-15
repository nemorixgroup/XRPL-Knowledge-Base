# Base58 Codec

**Phase:** 1 - Cryptographic Fundamentals  
**Status:** ✅ Done (`0.0.2-dev`)  
**Implementation:** [`lib/src/codec/xrpl_base58.dart`](https://github.com/nemorixgroup/xrpl-flutter-sdk/blob/main/lib/src/codec/xrpl_base58.dart)  

## What This Is

XRPL encodes seeds and addresses as human-readable strings using
base58 - a text encoding similar to Bitcoin's, but with its own
alphabet and, for seeds/addresses, a checksum to detect corruption or
typos.

## Decision Taken

Implemented `XrplBase58` with four functions, deliberately split into
two pairs rather than one combined encode/decode:

```dart
static String encodeRaw(Uint8List bytes)
static Uint8List decodeRaw(String encoded)

static String encodeWithChecksum(Uint8List bytes)
static Uint8List decodeWithChecksum(String encoded)
```

## Why XRPL's Own Alphabet

XRPL does not use Bitcoin's base58 alphabet. It rearranges the same 58
characters into a different order:

```
Bitcoin: 123456789ABCDEFGHJKLMNPQRSTUVWXYZabcdefghijkmnopqrstuvwxyz
XRPL:    rpshnaf39wBUDNEGHJKLM4PQRST7VWXYZ2bcdeCg65jkm8oFqi1tuvAxyz
```

This is a hard requirement, not a stylistic choice - decoding an XRPL
seed with Bitcoin's alphabet (or vice versa) produces completely wrong
bytes. The alphabet and the reference behavior are taken directly from
the `ripple-address-codec` implementation used across official
Ripple/XRPL client libraries.

## Why Raw and Checksummed Versions Are Separate

Not everything encoded in XRPL base58 needs a checksum, and the
checksummed functions are built **on top of** the raw ones rather than
duplicating the conversion logic:

```dart
static String encodeWithChecksum(Uint8List bytes) {
  final checksum = checksumOf(bytes);
  final withChecksum = Uint8List.fromList([...bytes, ...checksum]);
  return encodeRaw(withChecksum); // reuses encodeRaw
}
```

## The Checksum: Why and How

The checksum is the first 4 bytes of `SHA256(SHA256(bytes))` (the same
`Base58Check` scheme Bitcoin uses), computed via `pointycastle`'s
`SHA256Digest`.

Its purpose is safety, not compression: it lets `decodeWithChecksum`
detect a single mistyped character and reject it explicitly, instead
of silently returning corrupted bytes that could later be used to
derive a wrong key or address.

```dart
static Uint8List decodeWithChecksum(String encoded) {
  final allBytes = decodeRaw(encoded);
  final data = allBytes.sublist(0, allBytes.length - 4);
  final receivedChecksum = allBytes.sublist(allBytes.length - 4);
  final expectedChecksum = checksumOf(data);
  // mismatch -> XrplCryptoException, never a silent pass-through
}
```

## Leading Zero Bytes

Base58 cannot numerically represent a leading zero (`0x00...` becomes
indistinguishable from a shorter number once converted). Both
`encodeRaw` and `decodeRaw` handle this explicitly: each leading zero
byte becomes a leading `alphabet[0]` character (`'r'`) on encode, and
is restored symmetrically on decode.

## Error Handling

- `decodeRaw` throws `XrplCryptoException` on any character outside
  the XRPL alphabet, naming the character and its position
- `decodeWithChecksum` throws on a checksum mismatch, on input too
  short to contain a checksum (fewer than 4 bytes), or by propagating
  `decodeRaw`'s invalid-character error

## Test Vectors / Verification

- **Checksum correctness:** validated against a double-SHA256 value
  computed independently via Python's `hashlib`, not just a
  structural "4 bytes" assertion
- **Round-trip:** `encodeRaw`/`decodeRaw` tested against multiple byte
  patterns, including leading zeros, recovering the exact original
  bytes
- **Corruption detection:** a dedicated test flips the last character
  of a valid checksummed string (simulating a real mistyped character)
  and asserts `decodeWithChecksum` rejects it

43 unit tests total across four test files. See
[`test/codec/`](https://github.com/nemorixgroup/xrpl-flutter-sdk/tree/main/test/codec).

## Official Sources

- [`ripple-address-codec`](https://github.com/XRPLF/xrpl.js/tree/main/packages/ripple-address-codec) - alphabet and Base58Check reference implementation
- [xrpl.org - Cryptographic Keys](https://xrpl.org/docs/concepts/accounts/cryptographic-keys)
