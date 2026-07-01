# Module 03 - XRP Token
 
> A deep technical look at XRP, the native digital asset of the XRP Ledger: its fixed supply model, the drop denomination, transaction fee mechanics, Ripple's escrow system, and the multiple protocol-level roles XRP plays across the ledger.
 
 
## What is XRP
 
XRP is the native digital asset of the XRP Ledger, an open-source, permissionless, and decentralized blockchain technology. Created in 2012 specifically for payments, XRP can settle transactions on the ledger in 3-5 seconds.
 
XRP can be sent directly without needing a central intermediary, making it a convenient instrument for bridging two different currencies quickly and efficiently. It is freely exchanged on the open market and used in the real world for enabling cross-border payments and microtransactions.
 
**Source**: [xrpl.org/about/xrp](https://xrpl.org/about/xrp)
 
 
## Fixed Supply: Pre-Mined at Genesis
 
This is one of the most important and differentiating properties of XRP.
 
There is a finite amount of XRP. All XRP already exists today. No more than the original 100 billion can ever be created. All XRP was pre-mined when the XRP Ledger launched in 2012, meaning the entire supply came into existence at once.
 
| Asset | Maximum Supply | How it is created |
|-------|---------------|-------------------|
| **XRP** | 100,000,000,000 (fixed) | Pre-mined at genesis in 2012, no new creation possible |
| **Bitcoin** | 21,000,000 | Miners receive new BTC per block until ~2140 |
| **Ethereum** | No fixed cap | Validators receive new ETH as staking rewards |
 
This rigidity is a deliberate design decision. There are no miners selling newly created XRP to cover operational costs, which eliminates a constant source of sell pressure that exists in Bitcoin.
 
**Source**: [xrpl.org/about/xrp](https://xrpl.org/about/xrp), [xrpl.org/docs/concepts/tokens](https://xrpl.org/docs/concepts/tokens)
 
 
## Drops: The Smallest Unit of XRP
 
XRP has fixed precision to 6 decimal places and is represented as integer drops, where 1 million drops equals 1 XRP.
 
```
1 XRP = 1,000,000 drops
1 drop = 0.000001 XRP (the smallest possible unit)
```
 
All fees, reserves, and internal protocol values are expressed in drops. This matters when reading raw ledger data or working with the XRPL APIs.
 
**Source**: [xrpl.org/docs/concepts/tokens](https://xrpl.org/docs/concepts/tokens)
 
 
## Transaction Fees: Burned, Not Redistributed
 
This is a critical point that is widely misunderstood.
 
To protect the XRP Ledger from being disrupted by spam and denial-of-service attacks, each transaction must destroy a small amount of XRP. The current minimum transaction cost required by the network for a standard transaction is 0.00001 XRP (10 drops). It sometimes increases due to higher than usual load.
 
The transaction cost is not paid to any party. The XRP is irrevocably destroyed.
 
The standard ledger base fee is typically 10 drops, occasionally increased due to high volume. If validators vote to increase or lower the base fee, costs based on the standard fee are adjusted accordingly.
 
This means:
 
- Fees do not go to validators (unlike Ethereum)
- Fees do not go to Ripple or any other entity
- Fees are permanently removed from the total supply with every transaction

**Source**: [xrpl.org/docs/concepts/transactions/transaction-cost](https://xrpl.org/docs/concepts/transactions/transaction-cost)
 
 
## Ripple's Escrow: Programmable Supply Management
 
### Context
 
The XRPL founders gifted 80 billion XRP to Ripple. To provide predictability to the XRP supply, Ripple has locked 55 billion XRP (55% of the total possible supply) into a series of escrows using the XRP Ledger itself. The XRPL's transaction processing rules, enforced by the consensus protocol, control the release of the XRP.
 
As of October 2024, approximately 38 billion XRP remained in escrow.
 
**Source**: [xrpl.org/about/xrp](https://xrpl.org/about/xrp)
 
### How Escrow Works on XRPL
 
The XRP Ledger takes escrow a step further by replacing a third-party with an automated system built into the ledger itself. An escrow locks up XRP, which cannot be used or destroyed until specific conditions are met.
 
Ripple's escrow operates as follows in practice:
 
```
Every month:
- Up to 1,000,000,000 XRP is released from escrow
- Ripple uses what it needs for operations
- The remainder is re-locked into new escrow contracts
```
 
The escrow is hard-coded into XRPL. There is no backdoor that allows Ripple to access those locked coins outside the set maturation schedule. The entire lifecycle of escrow accounts and locked funds is publicly inspectable on-chain by anyone.
 
**Source**: [xrpl.org/docs/concepts/payment-types/escrow](https://xrpl.org/docs/concepts/payment-types/escrow), [xrpl.org/about/xrp](https://xrpl.org/about/xrp)
 
 
## The Multiple Protocol-Level Roles of XRP
 
This is where XRP differs most radically from other digital assets. XRP is not just a payment medium; it is the central element of the entire ledger infrastructure.
 
All accounts must hold at least a small amount of XRP in reserve and must burn small amounts of XRP as "gas" to pay for sending transactions.
 
The roles of XRP in the protocol are:
 
```
1. ACCOUNT RESERVE
   Every account requires a minimum XRP balance
   locked in the ledger to exist at all
 
2. TRANSACTION FEE
   Every operation burns a small amount of XRP
   (permanently destroyed, never redistributed)
 
3. DEX BRIDGE CURRENCY
   XRP acts as a bridge currency in the native DEX
   for currency pairs that have no direct liquidity
 
4. OFFER RESERVE
   Each active Offer in the DEX locks additional XRP
   as a reserve while the order is open
 
5. TRUST LINE RESERVE
   Each open Trust Line locks additional XRP
   as a reserve for that relationship
 
6. AMM LIQUIDITY
   XRP can be one side of any AMM pool
   natively in the protocol
```
 
**Source**: [xrpl.org/docs/concepts/tokens](https://xrpl.org/docs/concepts/tokens), [xrpl.org/about/xrp](https://xrpl.org/about/xrp)
 
 
## Performance Comparison: XRP vs Bitcoin
 
| Property | XRP | Bitcoin |
|----------|-----|---------|
| Settlement time | 3-5 seconds | +500 seconds |
| Cost per transaction | ~$0.0002 | ~$0.50 |
| Transactions per second | 1,500 TPS | 3 TPS |
| Energy consumption | Negligible | 0.3% of global energy use |
 
**Source**: [xrpl.org/about/xrp](https://xrpl.org/about/xrp)
 
 
## The Deflationary Mechanism
 
The combination of a fixed supply and permanently burned transaction fees makes XRP technically deflationary over time.
 
On the XRPL, transaction fees are destroyed by the ledger and are not paid to validators or any entity. XRPL burns are technical and continuous; they are not primarily a market promotion program. Fees are removed rather than redistributed.
 
This contrasts with Bitcoin, where miners receive fees as income, and with Ethereum, where validators partially accumulate fees. In XRPL, every transaction permanently reduces the total supply and no one receives those XRP.
 
**Source**: [xrpl.org/docs/concepts/transactions/transaction-cost](https://xrpl.org/docs/concepts/transactions/transaction-cost)
 
 
## Connection to Other Modules
 
- **Account reserves** (Module 05) are XRP locked in the state tree of the ledger per account
- **Trust Line reserves** (Module 05) are additional XRP locked for each open trust relationship
- **Offer reserves** (Module 04) are XRP locked by each active order in the DEX
- **XRP as bridge currency in the DEX** (Module 04) is the technical foundation of pathfinding and auto-bridging

 
## Official Sources
 
| Resource | URL |
|----------|-----|
| XRP Overview | https://xrpl.org/about/xrp |
| Transaction Cost | https://xrpl.org/docs/concepts/transactions/transaction-cost |
| Escrow | https://xrpl.org/docs/concepts/payment-types/escrow |
| Tokens Overview | https://xrpl.org/docs/concepts/tokens |
| XRPL FAQ | https://xrpl.org/about/faq |
| Ledger Explorer (Mainnet) | https://livenet.xrpl.org |
 
---
 
*Last updated: June 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
