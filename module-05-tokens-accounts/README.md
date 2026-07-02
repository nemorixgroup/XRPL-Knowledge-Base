# Module 05 - Tokens & Accounts
 
> A deep technical look at how accounts work on the XRP Ledger, the reserve system that governs XRP locking, the Trust Line model for tokens, and the three token standards available on XRPL: Trust Line Tokens, Multi-Purpose Tokens (MPTs), and Non-Fungible Tokens (NFTs).
 
 
## What is an Account on XRPL
 
An "Account" in the XRP Ledger is somewhere between the financial usage (like "bank account") and the computing usage (like "UNIX account"). Non-XRP currencies and assets are not stored in an XRP Ledger Account itself; each such asset is stored in an accounting relationship called a "Trust Line" that connects two parties.
 
**Source**: [xrpl.org/docs/concepts/accounts](https://xrpl.org/docs/concepts/accounts)
 
### How an Account is Created
 
There is not a dedicated "create account" transaction. The Payment transaction automatically creates a new account if the payment sends enough XRP to a mathematically valid address that does not already have an account. This is called funding an account, and creates an AccountRoot entry in the ledger. No other transaction can create an account.
 
```
To create an account on XRPL:
 
1. Someone sends a Payment with enough XRP
   to a valid address without an account
 
2. The protocol automatically creates
   an AccountRoot in the State Tree
 
3. The account is now active and can
   send transactions
```
 
### What an Account Contains
 
An account's core data includes: an XRP balance (some of which is set aside for the Reserve), a sequence number that helps ensure transactions are applied in the correct order and only once, a history of transactions that affected the account, and one or more ways to authorize transactions, including a master key pair, a "regular" key pair that can be rotated, and a signer list for multi-signing.
 
**Source**: [xrpl.org/docs/concepts/accounts](https://xrpl.org/docs/concepts/accounts)
 
 
## Reserves: XRP Locked by Ledger Usage
 
The reserve system is one of the most important and least understood mechanisms in XRPL.
 
### Base Reserve
 
The first time you receive XRP at your own XRP Ledger address, you must pay the account reserve (currently 1 XRP), which locks up that amount of XRP indefinitely.
 
### Owner Reserve
 
Each object that an account owns in the ledger increases its total reserve requirement by the owner reserve (currently 0.2 XRP per object).
 
Objects that count towards an owner's reserve requirement include: Checks, Deposit Preauthorizations, Escrows, NFT Offers, NFT Pages, Offers, Oracles, Payment Channels, Signer Lists, Tickets, and Trust Lines.
 
```
Example reserve calculation:
 
Base Reserve:                     1.0 XRP  (to exist)
Trust Line USD.Bitstamp:   +      0.2 XRP
Trust Line EUR.Gatehub:    +      0.2 XRP
1 active Offer on the DEX: +      0.2 XRP
 
Total locked:                     1.6 XRP
(cannot be spent, but released when objects are removed)
```
 
When objects are removed from the ledger, they no longer count against the reserve requirement, and that XRP becomes available again.
 
**Source**: [xrpl.org/docs/concepts/accounts/reserves](https://xrpl.org/docs/concepts/accounts/reserves)
 
 
## Trust Lines: The Bilateral Trust Model
 
### What They Are
 
A Trust Line is a bilateral relationship between two accounts that states: "I am willing to hold up to X units of the token issued by this account."
 
Trust lines are ledger objects that require a reserve of 0.2 XRP each. To help new users get started, the reserve amounts are waived for the first 2 trust lines created for a new account.
 
### How It Works in Practice
 
```
To receive USD.Bitstamp for the first time:
 
1. Your account sends a TrustSet transaction
   indicating you trust Bitstamp as a USD issuer,
   up to a configured limit
 
2. A RippleState object is created in the ledger
   connecting your account with Bitstamp
 
3. You can now receive and hold
   USD issued by Bitstamp
 
4. Your owner reserve increases by 0.2 XRP
```
 
### Trust Line as a Ledger Object
 
A RippleState ledger entry represents a trust line between two accounts. Each account can change its own settings with a TrustSet transaction. A trust line that is entirely in its default state is considered the same as a trust line that does not exist and is automatically deleted.
 
**Source**: [xrpl.org/docs/concepts/tokens/fungible-tokens/trust-line-tokens](https://xrpl.org/docs/concepts/tokens/fungible-tokens/trust-line-tokens), [xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate](https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate)
 
### How a Token is Identified on XRPL
 
Every token on XRPL is identified by a combination of currency code and issuer. For example, to represent $153.75 US dollars issued by account r9cZA1...:
 
```json
{
  "currency": "USD",
  "value": "153.75",
  "issuer": "r9cZA1mLK5R5Am25ArfXFmqgNwjZgnfk59"
}
```
 
This means `USD.Bitstamp` and `USD.Gatehub` are two completely different tokens even though both have the currency code "USD". The issuer is what distinguishes them.
 
**Source**: [xrpl.org/docs/references/protocol/data-types/currency-formats](https://xrpl.org/docs/references/protocol/data-types/currency-formats)
 
 
## The Three Token Standards on XRPL
 
XRPL supports three token standards, each serving a different purpose.
 
**Source**: [xrpl.org/docs/concepts/tokens](https://xrpl.org/docs/concepts/tokens)
 
### Trust Line Tokens (v1)
 
Trust Line Tokens are the "version 1" fungible token standard. They are fully available in production on the XRP Ledger and can be used for cross-currency payments or traded in the decentralized exchange.
 
Typical use cases:
- Stablecoins (USD.Bitstamp, EUR.Gatehub)
- Representation of real-world assets
- Community credit systems
- Tokens for the native DEX
Source: [xrpl.org/docs/concepts/tokens/fungible-tokens/trust-line-tokens](https://xrpl.org/docs/concepts/tokens/fungible-tokens/trust-line-tokens)
 
### Multi-Purpose Tokens (MPTs, v2)
 
Multi-Purpose Tokens are a form of fungible token on the XRP Ledger designed for greater efficiency and ease of use, based on lessons learned from trust line tokens. MPTs use a unidirectional model with an explicit separation between issuer and holder, and intentionally lack features like rippling to keep the conceptual model simpler.
 
Key differences from Trust Line Tokens:
 
| Aspect | Trust Line Tokens | MPTs |
|--------|------------------|------|
| Model | Bidirectional (two accounts) | Unidirectional (issuer to holder) |
| Rippling | Yes (configurable) | No (intentionally excluded) |
| On-chain metadata | No | Yes (up to 1024 bytes) |
| Configurable supply cap | No | Yes |
| Storage efficiency | Standard | More efficient |
| Best for | DEX, legacy, community credit | Stablecoins, RWA, institutional use |
 
For most new tokens, MPTs are preferred. Specific cases where trust line tokens may be preferred include: DEX compatibility requirements, community credit use cases, compatibility with legacy software, and use cases requiring representation of very large and very small quantities of the same token.
 
**Source**: [xrpl.org/docs/concepts/tokens/fungible-tokens/multi-purpose-tokens](https://xrpl.org/docs/concepts/tokens/fungible-tokens/multi-purpose-tokens), [xrpl.org/docs/concepts/tokens/fungible-tokens](https://xrpl.org/docs/concepts/tokens/fungible-tokens)
 
### Non-Fungible Tokens (NFTs)
 
The XRP Ledger natively supports non-fungible tokens. Non-fungible tokens serve to encode ownership of unique physical, non-physical, or purely digital goods, such as works of art or in-game items.
 
On the XRP Ledger, an NFT is represented as an NFToken object. An NFT is a unique, indivisible unit that is not used for payments. Users can mint (create), hold, buy, sell, and burn (destroy) NFTs. The ledger stores up to 32 NFTs owned by the same account in a single NFTokenPage object to save space. As a result, the owner reserve for NFTs only increases when the ledger needs to make a new page to store additional tokens.
 
**Source**: [xrpl.org/docs/concepts/tokens/nfts](https://xrpl.org/docs/concepts/tokens/nfts)
 
 
## Rippling: Automatic Settlement Through Trust Lines
 
Rippling is a property of Trust Line Tokens that is important to understand before building on XRPL.
 
The XRP Ledger can automatically and atomically use trust line debts to settle payments through rippling.
 
```
Alice holds 10 USD.Bitstamp  (trust line with Bitstamp)
Bob   holds 10 USD.Bitstamp  (trust line with Bitstamp)
 
Carol wants to send 5 USD to Bob, routed through Alice:
  Carol -> Alice -> Bob
 
The protocol "ripples" through Alice:
  - Alice loses 5 USD on her trust line with Carol
  - Alice gains 5 USD on her trust line with Bob
  (Alice's net balance is unchanged)
```
 
MPTs deliberately exclude rippling to simplify the token model.
 
**Source**: [xrpl.org/docs/concepts/tokens](https://xrpl.org/docs/concepts/tokens)
 
 
## Summary: XRP vs Trust Line Token vs MPT
 
| Property | XRP | Trust Line Token | MPT |
|----------|-----|-----------------|-----|
| Requires Trust Line to receive | No | Yes | Yes (MPT entry) |
| Issuer identified on-chain | N/A | Yes (issuer + currency) | Yes (MPT Issuance ID) |
| Rippling | N/A | Yes (configurable) | No |
| Supply cap | Fixed at 100B | No (configurable) | Yes (configurable) |
| On-chain metadata | No | No | Yes (up to 1024 bytes) |
| Available in DEX | Yes | Yes | Yes (with MPT DEX amendment) |
| Transfer fee | No | Yes (configurable) | Yes (configurable) |
 
**Source**: [xrpl.org/docs/references/protocol/data-types/currency-formats](https://xrpl.org/docs/references/protocol/data-types/currency-formats)
 
 
## Connection to Other Modules
 
- Every Trust Line and every active Offer on the DEX (Module 04) locks 0.2 XRP as owner reserve (Module 03)
- Trust Line Tokens are the IOUs referenced in the context of pathfinding and auto-bridging (Module 04)
- The DEX auto-bridging mechanism requires tokens to exist as Trust Lines in the ledger before they can be traded
- MPTs are the foundation for new institutional and real-world asset use cases covered in Module 07

 
## Official Sources
 
| Resource | URL |
|----------|-----|
| Accounts | https://xrpl.org/docs/concepts/accounts |
| Reserves | https://xrpl.org/docs/concepts/accounts/reserves |
| Tokens Overview | https://xrpl.org/docs/concepts/tokens |
| Trust Line Tokens | https://xrpl.org/docs/concepts/tokens/fungible-tokens/trust-line-tokens |
| Authorized Trust Lines | https://xrpl.org/docs/concepts/tokens/fungible-tokens/authorized-trust-lines |
| Fungible Tokens | https://xrpl.org/docs/concepts/tokens/fungible-tokens |
| Multi-Purpose Tokens | https://xrpl.org/docs/concepts/tokens/fungible-tokens/multi-purpose-tokens |
| Non-Fungible Tokens | https://xrpl.org/docs/concepts/tokens/nfts |
| RippleState (Trust Line format) | https://xrpl.org/docs/references/protocol/ledger-data/ledger-entry-types/ripplestate |
| Currency Formats | https://xrpl.org/docs/references/protocol/data-types/currency-formats |
| Known Amendments | https://xrpl.org/resources/known-amendments |
| Ledger Explorer (Mainnet) | https://livenet.xrpl.org |
 
---
 
*Last updated: July 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
