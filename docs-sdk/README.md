# SDK Technical Decisions

This section documents the implementation decisions, technical
rationale, and key references behind **xrpl_flutter_sdk**, the first
native Flutter/Dart SDK for the XRP Ledger.

Every decision documented here is grounded in official XRPL sources
(xrpl.org, the `xrpl-keypairs` and `ripple-address-codec` reference
implementations, and RippleX documentation). Where relevant,
implementation code and test vectors are included to make each
decision fully verifiable.

## Why This Documentation Exists

Building a production-quality SDK for XRPL requires making decisions
that are not always obvious - which cryptographic library to use, why
a specific encoding standard was chosen, how an algorithm was
verified against the official specification.

This documentation answers those questions for:

- **Contributors** who want to understand the rationale before
  submitting a pull request
- **Developers** who want to verify that the SDK follows official
  XRPL standards
- **Auditors** who need a clear trail from specification to
  implementation

## Structure

```
docs-sdk/
  README.md          <- You are here
  phase-1/            <- Cryptographic Fundamentals
  phase-2/            <- Addresses
  phase-3/            <- Connection Layer
  phase-4/            <- Core Transactions
  phase-5/            <- DEX & Cross-Currency
  phase-6/            <- Conditionals & Channels
  phase-7/            <- Tokenization
  phase-8/            <- Account Security & Compliance
```

## Phase 1 - Cryptographic Fundamentals

| Feature | Description | Status |
|---------|--------------|--------|
| [Entropy](phase-1/entropy/README.md) | Secure random entropy generation for XRPL seeds (16 bytes) | ✅ Done |
| [Base58 Codec](phase-1/base58-codec/README.md) | XRPL's own base58 alphabet, encode/decode, Base58Check-style checksum | ✅ Done |
| Family Seed Encoding | Combining entropy + codec into an XRPL seed (`s...` prefix) | 🔄 Next |
| secp256k1 Key Derivation | Root key, intermediate key, curve point derivation | ⏳ Pending |
| Ed25519 Key Derivation | Curve derivation, `0xED` public key prefix | ⏳ Pending |
| XrplWallet Unified API | Public API unifying both algorithms | ⏳ Pending |
| Error Handling & Validation | Cryptographic exception design across the SDK | ⏳ Pending |
| Test Suite & Official Vectors | Full Phase 1 suite validated against reference vectors | ⏳ Pending |

## Phase 2 - Addresses

| Feature | Description | Status |
|---------|--------------|--------|
| Classic Address | SHA-256 + RIPEMD160 derivation, `r...` format | ⏳ Pending |
| X-Address | Network + destination tag + address encoding | ⏳ Pending |

## Phase 3 - Connection Layer

| Feature | Description | Status |
|---------|--------------|--------|
| WebSocket/JSON-RPC Client | Transport design, request/response model | ⏳ Pending |
| Network Environments | Mainnet/Testnet/Devnet configuration | ⏳ Pending |
| Account & Ledger Queries | `account_info`, `server_info`, subscribe streams | ⏳ Pending |

## Phase 4 - Core Transactions

| Feature | Description | Status |
|---------|--------------|--------|
| Transaction Model | Payment, TrustSet structure and autofill | ⏳ Pending |
| Signing | Single-signing flow, `SigningPubKey`/`TxnSignature` | ⏳ Pending |
| Submission | `submit`, `submitAndWait`, validation lifecycle | ⏳ Pending |

## Phase 5 - DEX & Cross-Currency

| Feature | Description | Status |
|---------|--------------|--------|
| OfferCreate/OfferCancel | Order book mechanics | ⏳ Pending |
| AMM | AMMCreate/Deposit/Withdraw/Vote/Bid | ⏳ Pending |
| Path Finding | Cross-currency payment routing | ⏳ Pending |

## Phase 6 - Conditionals & Channels

| Feature | Description | Status |
|---------|--------------|--------|
| Escrow | Create/Finish/Cancel, crypto-conditions | ⏳ Pending |
| Payment Channels | Create/Fund/Claim | ⏳ Pending |
| Checks | Create/Cash/Cancel | ⏳ Pending |

## Phase 7 - Tokenization

| Feature | Description | Status |
|---------|--------------|--------|
| NFTs | Mint/Burn/CreateOffer/AcceptOffer/CancelOffer | ⏳ Pending |
| Multi-Purpose Tokens (MPT) | Issuance and transfer model | ⏳ Pending |
| Clawback | Issuer recovery mechanics | ⏳ Pending |

## Phase 8 - Account Security & Compliance

| Feature | Description | Status |
|---------|--------------|--------|
| Multi-Signing | SignerListSet, quorum model | ⏳ Pending |
| Regular Key | Key rotation without changing the address | ⏳ Pending |
| Tickets | TicketCreate, out-of-order sequencing | ⏳ Pending |
| DepositPreauth / Credentials | Compliance-oriented access control | ⏳ Pending |

## Related

- [xrpl_flutter_sdk](https://github.com/nemorixgroup/xrpl-flutter-sdk) - the SDK itself
- [XRPL Knowledge Base](../README.md) - official source documentation
