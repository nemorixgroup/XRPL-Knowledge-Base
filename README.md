# XRPL Knowledge Base
 
> A comprehensive, in-depth technical guide to the XRP Ledger (XRPL) - covering its architecture, consensus protocol, native services, token economics, decentralized finance primitives, and development ecosystem.
 
## About This Repository
 
This repository is a structured knowledge base built for developers, researchers, and engineers who want to understand the XRP Ledger at a deep technical level.
 
The content is organized in progressive modules, starting from foundational concepts and advancing toward hands-on development. Each module is self-contained and can be studied independently, but reading them in order is recommended for a complete understanding.
 
**Who is this for?**
 
- Developers exploring the XRP Ledger for the first time
- Engineers evaluating XRPL for payments, tokenization, or DeFi projects
- Blockchain enthusiasts who want to understand how XRPL differs from other networks
- Anyone building financial applications on open, decentralized infrastructure
  
**Languages:** English (primary).
 
## Repository Structure
 
```
xrpl-knowledge-base/
- README.md                              <- You are here (General Index)
- module-01-foundations/
    - README.md                          <- History, origin, Ripple Labs, mission
- module-02-ledger-consensus/
    - README.md                          <- Ledger structure, RPCA, validators, UNL
- module-03-xrp-token/
    - README.md                          <- Tokenomics, supply, fees, escrow, burning
- module-04-defi-native/
    - README.md                          <- DEX, Offers, Orderbook, AMMs, Pathfinding
- module-05-tokens-accounts/
    - README.md                          <- Accounts, Reserves, Trust Lines, IOUs, NFTs
- module-06-development/
    - README.md                          <- SDKs, transactions, Testnet, dev tools
- module-07-ecosystem/
    - README.md                          <- Use cases, adoption, comparisons, roadmap
```
 
## Modules
 
| # | Module | Topics |
|---|--------|--------|
| 01 | [Foundations & History](./module-01-foundations/README.md) | Origin, founders, Ripple Labs, open-source mission |
| 02 | [Ledger & Consensus](./module-02-ledger-consensus/README.md) | Ledger structure, RPCA, validators, UNL, finality |
| 03 | [XRP Token](./module-03-xrp-token/README.md) | Tokenomics, fixed supply, fees, escrow, burning |
| 04 | [Native DeFi](./module-04-defi-native/README.md) | DEX, Offers, Orderbook, AMMs, Pathfinding, Auto-Bridging |
| 05 | [Tokens & Accounts](./module-05-tokens-accounts/README.md) | Accounts, Reserves, Trust Lines, IOUs, NFTs, MPTs |
| 06 | [Development on XRPL](./module-06-development/README.md) | SDKs, transaction types, Testnet, dev tools |
| 07 | [Ecosystem & Future](./module-07-ecosystem/README.md) | Use cases, institutional adoption, comparisons, roadmap |
 
## Official Resources
 
### Core
 
| Resource | URL |
|----------|-----|
| Official Website | https://xrpl.org |
| Developer Documentation | https://xrpl.org/docs |
| Concepts Reference | https://xrpl.org/docs/concepts |
| History | https://xrpl.org/about/history |
| XRPL Overview | https://xrpl.org/about |
 
### Developer Tools
 
| Resource | URL |
|----------|-----|
| XRP Ledger Explorer (Mainnet) | https://livenet.xrpl.org |
| XRP Faucet (Testnet/Devnet) | https://xrpl.org/resources/dev-tools/xrp-faucets |
| WebSocket API Tool | https://xrpl.org/resources/dev-tools/websocket-api-tool |
| Transaction Sender (Testnet) | https://xrpl.org/resources/dev-tools/tx-sender |
| XRPL Learning Portal | https://learn.xrpl.org |
 
### SDKs (Official)
 
| Language | Repository |
|----------|------------|
| JavaScript / TypeScript | https://github.com/XRPLF/xrpl.js |
| Python | https://github.com/XRPLF/xrpl-py |
| Java | https://github.com/XRPLF/xrpl4j |
| Go | https://github.com/XRPLF/xrpl-go |
| PHP | https://github.com/XRPLF/XRPL-PHP |
 
### GitHub Organizations
 
| Organization | URL |
|--------------|-----|
| XRPL Foundation (core) | https://github.com/XRPLF |
| Ripple (company) | https://github.com/ripple |
| Official Dev Portal | https://github.com/XRPLF/xrpl-dev-portal |
 
### Community & News
 
| Resource | URL |
|----------|-----|
| XRPL Blog | https://xrpl.org/blog |
| XRPL Discord | https://discord.gg/sfX3ERAMjH |
| XRPL Foundation | https://xrpl.foundation |
| XRPL Grants | https://xrplgrants.org |
| GitHub Discussions | https://github.com/XRPLF/xrpl-dev-portal/discussions |
 
## Key Technical Facts (Quick Reference)
 
| Property | Value |
|----------|-------|
| Initial Release | June 2012 |
| Consensus Algorithm | XRP Ledger Consensus Protocol (RPCA) |
| Transaction Finality | 3-5 seconds |
| Throughput | ~1,500 TPS (sustained) |
| Minimum Transaction Fee | 10 drops (0.00001 XRP) |
| Native Token | XRP |
| Total Supply | 100,000,000,000 XRP (fixed, pre-mined) |
| Energy Usage | No mining - negligible energy consumption |
| Native DEX | Yes - built into the protocol since 2012 |
| Native AMMs | Yes - integrated via XLS-30 amendment (2024) |
| Open Source | Yes (ISC License) |
| Core Implementation | rippled (C++) |
 
## Why XRPL?
 
The XRP Ledger was designed from the ground up for payments - fast, low-cost, and energy-efficient. Several properties make it technically distinctive:
 
**No mining.** The XRP Ledger uses the XRP Ledger Consensus Protocol (RPCA), where a network of validators agrees on transaction ordering without proof-of-work or proof-of-stake. This eliminates energy waste and enables deterministic finality in 3-5 seconds.
 
**Native DEX.** Unlike other blockchains where exchanges are external applications or smart contracts, XRPL's decentralized exchange is part of the protocol itself - operating continuously since 2012. Orders live in the ledger's state data and are matched deterministically on every ledger close.
 
**Native AMMs.** Since the XLS-30 amendment in 2024, XRPL also includes Automated Market Makers integrated natively into the protocol, working alongside the order book DEX.
 
**Fixed supply.** All 100 billion XRP were created at genesis in 2012. There is no mining, no inflation, and no new issuance. Transaction fees are burned, making the supply deflationary over time.
 
**Trust Lines and IOUs.** XRPL supports a rich token model where any account can issue tokens representing any asset, tracked through trust lines - a mechanism built into the protocol since launch.
 
This is not just another blockchain. The XRP Ledger is purpose-built infrastructure where the asset, the exchange, and the settlement layer are the same unified system.
 
## Important Distinctions
 
| Concept | Description |
|---------|-------------|
| **XRP** | The native digital asset of the XRP Ledger |
| **XRPL** | The XRP Ledger - the open-source, decentralized network |
| **Ripple** | A private company that builds payment products on XRPL; it does not own or control the ledger |
| **XRPLF** | The XRP Ledger Foundation - a non-profit supporting the open-source ecosystem |
 
## About This Guide
 
This knowledge base is built from official XRP Ledger documentation, technical references published at xrpl.org, and verified on-chain data. No speculative content or price predictions are included.
 
The goal is to provide a clear, honest, and technically accurate resource for understanding the XRP Ledger from the inside out.

## Support This Project

If this project is useful to you or your team, consider supporting its development. Every contribution helps cover infrastructure, documentation, and the time invested in building and maintaining this open source resource for the XRPL community. Thank you!

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20A%20Coffee-Support-FFDD00?logo=buy-me-a-coffee&logoColor=black)](https://buymeacoffee.com/nemorixgroupllc)
[![Sponsor](https://img.shields.io/badge/Sponsor-GitHub-EA4AAA?logo=github-sponsors&logoColor=white)](https://github.com/sponsors/nemorixgroup)
[![Ko-fi](https://img.shields.io/badge/Ko--fi-Support-FF5E5B?logo=ko-fi&logoColor=white)](https://ko-fi.com/nemorixgroupllc) 

---
 
*Last updated: June 2026*
*Maintained by: [Miguel Fagundez](https://github.com/miguelfagundez) & [Nemorix Group, LLC](https://github.com/nemorixgroup)*
