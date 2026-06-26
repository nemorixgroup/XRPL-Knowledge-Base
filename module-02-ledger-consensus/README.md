# Module 02 - Ledger & Consensus
 
> A deep technical look at the structure of the XRP Ledger, how ledger versions are built and validated, the role of validators, the Unique Node List (UNL), and how the consensus protocol differs from Bitcoin and Ethereum.
 
 
## What is a Ledger
 
The XRP Ledger is a shared, append-only database maintained by thousands of servers worldwide. Every few seconds, all participating servers agree on a new version of that database. Once agreed upon, that version is sealed forever.
 
The XRP Ledger has a new ledger version every several seconds. When the network agrees on the contents of a ledger version, that ledger version is validated and its contents can never change. The validated ledger versions that preceded it form the ledger history.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
 
## The 3 Parts of a Ledger
 
A single ledger version consists of three parts: a **header**, a **transaction tree**, and a **state tree**.
 
```
┌─────────────────────────────────────────┐
│             LEDGER #94,000,000          │
├─────────────────────────────────────────┤
│  HEADER                                 │
│  - Ledger Index (position in chain)     │
│  - Hash (unique digital fingerprint)    │
│  - Close timestamp                      │
│  - Parent ledger hash                   │
├─────────────────────────────────────────┤
│  TRANSACTION TREE                       │
│  - Transactions applied to create       │
│    this ledger version                  │
├─────────────────────────────────────────┤
│  STATE TREE                             │
│  - All account balances                 │
│  - All account settings                 │
│  - All DEX orders (Offers)              │
│  - All AMM pools                        │
│  - All Trust Lines                      │
└─────────────────────────────────────────┘
```
 
### The Header
 
The ledger header contains the ledger index (its position in the chain), the ledger hash (a unique identifier of its contents such that any change produces a completely different hash), the parent ledger hash, the official close time when the contents were finalized, and cryptographic checksums of both the state data and transaction set.
 
Source: [xrpl.org/docs/concepts/ledgers/ledger-structure](https://xrpl.org/docs/concepts/ledgers/ledger-structure)
 
### The Transaction Tree
 
Transactions are the only way to authorize changes to an account or to anything else in the ledger. Each transaction is cryptographically signed by the account owner. Each ledger version contains only the transactions applied to the previous ledger version to create the new one. The metadata records the exact effects of each transaction on the ledger state data.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
### The State Tree
 
The state data is a set of ledger objects, also called ledger entries, that collectively represent all settings, balances, and relationships at a given point in time. To store or retrieve an object in the state data, the protocol uses that object's unique Ledger Object ID.
 
This is the most important part of the ledger. DEX orders, AMM pools, Trust Lines, and account balances all live here as ledger entries in the state tree.
 
Source: [xrpl.org/docs/references/protocol/ledger-data](https://xrpl.org/docs/references/protocol/ledger-data)
 
 
## The 3 States of a Ledger
 
At any given moment, the network has ledger versions in three distinct states:
 
```
OPEN           ->     CLOSED       ->    VALIDATED
(accepting           (undergoing          (sealed
transactions)        consensus)           forever)
```
 
Applications must only rely on information in validated ledgers, not on provisional results of candidate transactions. The results of a transaction become immutable only when that transaction is included in a validated ledger.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
 
## How a Ledger is Identified
 
Each ledger has two distinct identifiers with different purposes:
 
- The **ledger index** indicates the ledger's position in the chain, numbered incrementally.
- The **ledger hash** is a digital fingerprint of the ledger's contents. If any detail changes, the hash is completely different.
Ledgers from different chains can have the same ledger index but different hashes. The ledger index tells you where it is; the ledger hash proves what it contains.
 
Source: [xrpl.org/docs/concepts/ledgers/ledger-structure](https://xrpl.org/docs/concepts/ledgers/ledger-structure)
 
 
## The Consensus Process
 
### The Problem it Solves
 
Without a consensus mechanism, different servers could maintain different versions of the ledger and never agree. Consensus is the process that guarantees all servers reach the same result without a central authority.
 
### Network Participants
 
Servers in the network can be **tracking servers** or **validators**. Tracking servers distribute transactions from clients and respond to queries about the ledger. Validating servers do the same and additionally contribute to advancing the ledger history by sending validation messages.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
### The Consensus Rounds
 
Consensus is an iterative process in which servers relay proposals, meaning sets of candidate transactions. Servers communicate and update proposals until a supermajority of chosen validators agrees on the same set of candidate transactions.
 
The threshold increases across rounds: at the start of the first round, at least 50% of peers must agree. The final threshold for a completed consensus round is 80% agreement.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
### Validation
 
Validation is the second stage of the consensus process. It verifies that servers got the same results and declares a ledger version final. Each server independently calculates the new ledger by following the same deterministic rules:
 
1. Start with the previous validated ledger.
2. Place the agreed-upon transaction set in canonical order (hard to manipulate).
3. Process each transaction according to its instructions.
4. Update the ledger header with appropriate metadata.
5. Calculate the identifying hash of the new ledger version.
Validators relay their results as a signed message containing the hash of the ledger version they calculated. The network recognizes a ledger version as validated when a supermajority of peers has signed and broadcast the same validation hash.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
 
## The Unique Node List (UNL)
 
### What it is
 
A Unique Node List (UNL) is a server's list of validators that it trusts not to collude. Every XRP Ledger server is configured with a UNL, which determines which validation votes it listens to and which it discards during the consensus process.
 
By design, each entry in a UNL should represent an independent entity, such as a business, a university, another type of organization, or an individual. This prevents any single party from having outsized influence over the network.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/unl](https://xrpl.org/docs/concepts/consensus-protocol/unl)
 
### The Risk of Divergent UNLs
 
If two servers operate with completely different UNLs, they are likely to reach different conclusions about when ledgers are validated. This could lead to a fork in the network, where parties on different sides cannot agree on what has happened and cannot transact with one another.
 
To prevent this, by default XRP Ledger servers are configured to use validator lists maintained by the XRPL Foundation and Ripple, which ensures 100% overlap between servers using the same list.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/unl](https://xrpl.org/docs/concepts/consensus-protocol/unl), [xrpl.org/docs/concepts/consensus-protocol/consensus-protections](https://xrpl.org/docs/concepts/consensus-protocol/consensus-protections)
 
### Validators on the Network Today
 
There are 150+ validators on the network, with 35+ on the default Unique Node List. Ripple runs only 1 of those nodes.
 
Source: [xrpl.org/about/faq](https://xrpl.org/about/faq)
 
 
## XRPL vs Bitcoin vs Ethereum: Consensus Comparison
 
| Aspect | XRPL | Bitcoin | Ethereum |
|--------|------|---------|----------|
| Mechanism | RPCA (federated consensus) | Proof of Work | Proof of Stake |
| Finality | 3-5 seconds, immediate | 10-60 minutes (6 confirmations) | 12-15 minutes |
| Energy use | Negligible, no mining | Very high | Lower than PoW |
| New tokens created | None | Miners receive new BTC | Validators receive new ETH |
| Reorganization risk | Not possible after validation | Theoretically possible | Theoretically possible |
 
 
## Why Immediate Finality Matters
 
Once a ledger is validated in XRPL, it cannot be reorganized or reversed. This is a fundamental design property of the protocol called **immediate finality**.
 
Transactions with result code `tesSUCCESS` and `validated: true` have immutably succeeded. Transactions with other result codes and `validated: true` have immutably failed. There is no ambiguity and no waiting for additional confirmations.
 
This property is what makes the native DEX safe for arbitrageurs to operate on every ledger close without reorganization risk, which is explored in Module 04.
 
Source: [xrpl.org/docs/concepts/consensus-protocol/consensus-structure](https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure)
 
 
## Verify it Yourself
 
Visit [livenet.xrpl.org](https://livenet.xrpl.org) and observe in real time:
 
- The ledger index incrementing every 3-5 seconds
- The unique hash of each ledger
- The transactions included in each close
- The number of validators that signed each ledger

 
## Connection to Other Modules
 
- The **state tree** contains DEX orders (Offers) covered in Module 04
- The **state tree** contains AMM pools also covered in Module 04
- The **state tree** contains accounts and Trust Lines covered in Module 05
- **Immediate finality** is the technical foundation that allows the native DEX to operate safely without reorganization risk
- **Transactions** are the only mechanism to change anything in the ledger, including placing an order on the DEX or modifying an AMM pool

 
## Official Sources
 
| Resource | URL |
|----------|-----|
| Consensus Structure | https://xrpl.org/docs/concepts/consensus-protocol/consensus-structure |
| Consensus Protocol Overview | https://xrpl.org/docs/concepts/consensus-protocol |
| Unique Node List (UNL) | https://xrpl.org/docs/concepts/consensus-protocol/unl |
| Consensus Protections | https://xrpl.org/docs/concepts/consensus-protocol/consensus-protections |
| Ledger Structure | https://xrpl.org/docs/concepts/ledgers/ledger-structure |
| Ledger Data Formats | https://xrpl.org/docs/references/protocol/ledger-data |
| XRPL FAQ | https://xrpl.org/about/faq |
| Ledger Explorer (Mainnet) | https://livenet.xrpl.org |
 
---
 
*Last updated: June 2026*  
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
