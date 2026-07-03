# Module 06 - Development on XRPL
 
> A practical technical guide to building on the XRP Ledger: the official SDKs, the development model, the most important transaction types, network environments, dev tools, and the transaction lifecycle every developer needs to understand.
 
 
## The XRPL Development Model
 
Developing on XRPL is conceptually different from developing on Ethereum or Solana. There are no general-purpose smart contracts. All business logic is expressed through **native transaction types** that the protocol already knows and executes deterministically.
 
The XRP Ledger protocol itself handles the core financial operations: payments, trading, liquidity provision, escrow, token issuance, and more. The developer's job is to know which transaction type to use and how to configure it correctly.
 
The basic building blocks of XRP Ledger-based applications are: connecting to the XRP Ledger, creating or loading a wallet, submitting transactions to make changes, and querying the ledger to read state.
 
**Source**: [xrpl.org/docs/tutorials/get-started/get-started-javascript](https://xrpl.org/docs/tutorials/get-started/get-started-javascript)
 
 
## Official Client Libraries (SDKs)
 
| Language | Library | Official Repository |
|----------|---------|-------------------|
| JavaScript / TypeScript | xrpl.js | https://github.com/XRPLF/xrpl.js |
| Python | xrpl-py | https://github.com/XRPLF/xrpl-py |
| Java | xrpl4j | https://github.com/XRPLF/xrpl4j |
| Go | xrpl-go | https://github.com/XRPLF/xrpl-go |
| PHP | XRPL-PHP | https://github.com/XRPLF/XRPL-PHP |
 
xrpl.js is the recommended library for integrating a JavaScript/TypeScript application with the XRP Ledger, especially when using advanced functionality such as IOUs, payment paths, the decentralized exchange, account settings, payment channels, escrows, and multi-signing. It works in both Node.js (v20+) and modern web browsers.
 
**Source**: [xrpl.org/docs/references/client-libraries](https://xrpl.org/docs/references/client-libraries), [js.xrpl.org](https://js.xrpl.org/)
 
 
## The Basic Application Flow
 
Every XRP Ledger application follows the same four-step pattern:
 
```
1. CONNECT
   Connect to a public or self-hosted node
   via WebSocket or JSON-RPC
 
2. WALLET
   Load or generate a wallet
   (key pair: private key + address)
 
3. TRANSACT
   Build, sign, and submit transactions
 
4. QUERY
   Read ledger state
   (balances, offers, trust lines, etc.)
```
 
Minimal example in JavaScript:
 
```javascript
import xrpl from "xrpl"
 
// 1. Connect
const client = new xrpl.Client("wss://s.altnet.rippletest.net:51233/")
await client.connect()
 
// 2. Wallet (funded from Testnet faucet)
const { wallet } = await client.fundWallet()
console.log("Address:", wallet.address)
 
// 3. Query
const response = await client.request({
  command: "account_info",
  account: wallet.address,
  ledger_index: "validated"
})
console.log("Balance:", response.result.account_data.Balance)
 
// 4. Disconnect
await client.disconnect()
```
 
**Source**: [xrpl.org/docs/tutorials/get-started/get-started-javascript](https://xrpl.org/docs/tutorials/get-started/get-started-javascript)
 
 
## The Most Important Transaction Types
 
The XRP Ledger defines a fixed set of transaction types. Each one produces deterministic, protocol-enforced results.
 
Source: [xrpl.org/docs/references/protocol/transactions/types](https://xrpl.org/docs/references/protocol/transactions/types)
 
### Payment: Send Value
 
The most fundamental transaction. Can send XRP directly, or tokens through pathfinding.
 
```json
{
  "TransactionType": "Payment",
  "Account": "rSenderAddress...",
  "Destination": "rReceiverAddress...",
  "Amount": "1000000"
}
```
 
Amount is specified in drops: 1,000,000 drops = 1 XRP.
 
### TrustSet: Create or Modify a Trust Line
 
Required before an account can receive any token other than XRP.
 
```json
{
  "TransactionType": "TrustSet",
  "Account": "ra5nK24KXen9AHvsdFTKHSANinZseWnPcX",
  "LimitAmount": {
    "currency": "USD",
    "issuer": "rsP3mgGb2tcYUrxiLFiHJiQXhsziegtwBc",
    "value": "100"
  }
}
```
 
**Source**: [xrpl.org/docs/references/protocol/transactions/types/trustset](https://xrpl.org/docs/references/protocol/transactions/types/trustset)  
 
### OfferCreate: Place an Order on the DEX
 
```json
{
  "TransactionType": "OfferCreate",
  "Account": "ra5nK24KXen9AHvsdFTKHSANinZseWnPcX",
  "TakerGets": "6000000",
  "TakerPays": {
    "currency": "GKO",
    "issuer": "ruazs5h1qEsqpke88pcqnaseXdm6od2xc",
    "value": "2"
  }
}
```
 
`TakerGets` is what you offer to give. `TakerPays` is what you want to receive.
 
**Source**: [xrpl.org/docs/references/protocol/transactions/types/offercreate](https://xrpl.org/docs/references/protocol/transactions/types/offercreate)
 
### AMMCreate: Create an AMM Liquidity Pool
 
```json
{
  "TransactionType": "AMMCreate",
  "Account": "rCreatorAddress...",
  "Amount": "1000000000",
  "Amount2": {
    "currency": "USD",
    "issuer": "rIssuerAddress...",
    "value": "500"
  },
  "TradingFee": 500
}
```
 
**Source**: [xrpl.org/docs/references/protocol/transactions/types/ammcreate](https://xrpl.org/docs/references/protocol/transactions/types/ammcreate)
 
### Other Commonly Used Transaction Types
 
| Transaction | Purpose |
|-------------|---------|
| `OfferCancel` | Cancel an existing open Offer on the DEX |
| `AccountSet` | Configure account settings and flags |
| `EscrowCreate` | Lock XRP or tokens with time/condition-based release |
| `EscrowFinish` | Release escrowed funds when conditions are met |
| `NFTokenMint` | Create a new non-fungible token |
| `AMMDeposit` | Add liquidity to an existing AMM pool |
| `AMMWithdraw` | Remove liquidity from an AMM pool |
 
 
## Network Environments
 
XRPL provides three separate environments for development:
 
```
MAINNET (production)
  wss://xrplcluster.com        (community-maintained, recommended)
  wss://s1.ripple.com          (Ripple-maintained)
  wss://s2.ripple.com          (full history, Ripple-maintained)
  Uses real XRP. Real money.
 
TESTNET (integration testing)
  wss://s.altnet.rippletest.net:51233
  https://s.altnet.rippletest.net:51234  (JSON-RPC)
  Free test XRP via faucet.
  Mirrors Mainnet behavior.
  May be reset periodically.
 
DEVNET (experimental features)
  wss://s.devnet.rippletest.net:51233
  For testing amendments under development.
  Do not use for production applications.
```
 
**Source**: [xrpl.org/docs/tutorials/get-started/get-started-javascript](https://xrpl.org/docs/tutorials/get-started/get-started-javascript)
 
 
## Official Developer Tools
 
| Tool | URL | Purpose |
|------|-----|---------|
| XRP Faucet | xrpl.org/resources/dev-tools/xrp-faucets | Get free Testnet/Devnet XRP |
| WebSocket API Tool | xrpl.org/resources/dev-tools/websocket-api-tool | Test API calls interactively |
| Transaction Sender | xrpl.org/resources/dev-tools/tx-sender | Send test transactions |
| Ledger Explorer (Mainnet) | livenet.xrpl.org | View the live ledger |
| Ledger Explorer (Testnet) | testnet.xrpl.org | View Testnet transactions |
| XRPL Learning Portal | learn.xrpl.org | Official interactive courses |
| Code Samples | xrpl.org/resources/code-samples | Working code examples |
 
**Source**: [xrpl.org/docs](https://xrpl.org/docs)
 
 
## Transaction Lifecycle
 
Understanding this is critical for every XRPL developer:
 
```
1. BUILD
   Construct the transaction JSON object
   with all required fields
 
2. AUTOFILL
   The library fills optional fields automatically
   (Fee, Sequence, LastLedgerSequence)
 
3. SIGN
   The transaction is signed with the sender's
   private key (never leaves the client)
 
4. SUBMIT
   The signed blob is sent to a node
   via WebSocket or JSON-RPC
 
5. PROPAGATION
   The node relays the transaction
   to its peers across the network
 
6. CONSENSUS
   Validators include the transaction
   in the next candidate ledger
 
7. VALIDATION
   The ledger is validated and the transaction
   result becomes immutable (tesSUCCESS or error code)
 
8. CONFIRMATION
   The client verifies the result
   by querying the validated ledger
```
 
The critical rule for developers: **only trust results from validated ledgers**. A "pending" or "open ledger" result is not final.
 
 
## Recommended Pattern: LastLedgerSequence
 
This is the most important safety pattern when submitting transactions:
 
```javascript
// Always include LastLedgerSequence to guarantee
// the transaction expires if not included
// in a nearby ledger close
 
const prepared = await client.autofill({
  TransactionType: "Payment",
  Account: wallet.address,
  Amount: "1000000",
  Destination: "rDestinationAddress..."
})
 
// autofill automatically adds:
// - Fee (based on current network load)
// - Sequence (account's next sequence number)
// - LastLedgerSequence (current ledger + ~4)
 
const signed = wallet.sign(prepared)
const result = await client.submitAndWait(signed.tx_blob)
 
if (result.result.meta.TransactionResult === "tesSUCCESS") {
  console.log("Confirmed in validated ledger")
}
```
 
**Source**: [xrpl.org/docs/tutorials/get-started/get-started-javascript](https://xrpl.org/docs/tutorials/get-started/get-started-javascript)
 
 
## Connection to Other Modules
 
- Each `TrustSet` creates a Trust Line (Module 05) and locks 0.2 XRP as owner reserve (Module 03)
- Each `OfferCreate` creates an Offer object in the State Tree (Module 02) and locks 0.2 XRP (Module 03)
- Each `AMMCreate` initializes an AMM pool integrated with the CLOB DEX (Module 04)
- All transaction results are immutable once included in a validated ledger (Module 02)
- Transaction fees for every operation are permanently burned (Module 03)
  
 
## Official Sources
 
| Resource | URL |
|----------|-----|
| XRPL Documentation | https://xrpl.org/docs |
| Get Started (JavaScript) | https://xrpl.org/docs/tutorials/get-started/get-started-javascript |
| Get Started (Python) | https://xrpl.org/docs/tutorials/get-started/get-started-python |
| Get Started (Java) | https://xrpl.org/docs/tutorials/get-started/get-started-java |
| Get Started (Go) | https://xrpl.org/docs/tutorials/get-started/get-started-go |
| Client Libraries Reference | https://xrpl.org/docs/references/client-libraries |
| Transaction Types Reference | https://xrpl.org/docs/references/protocol/transactions/types |
| Payment Transaction | https://xrpl.org/docs/references/protocol/transactions/types/payment |
| TrustSet Transaction | https://xrpl.org/docs/references/protocol/transactions/types/trustset |
| OfferCreate Transaction | https://xrpl.org/docs/references/protocol/transactions/types/offercreate |
| AMMCreate Transaction | https://xrpl.org/docs/references/protocol/transactions/types/ammcreate |
| XRP Faucet | https://xrpl.org/resources/dev-tools/xrp-faucets |
| WebSocket API Tool | https://xrpl.org/resources/dev-tools/websocket-api-tool |
| Code Samples | https://xrpl.org/resources/code-samples |
| XRPL Learning Portal | https://learn.xrpl.org |
| Ledger Explorer (Mainnet) | https://livenet.xrpl.org |
| Ledger Explorer (Testnet) | https://testnet.xrpl.org |
 
---
 
*Last updated: June 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
