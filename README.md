# MultiPools Documentation

Technical documentation for the MultiPools protocol — a multichain token launchpad built on Uniswap v4 and LayerZero V2.

Website: https://multipools.trade
X: https://x.com/multipoolstrade

---

## Contents

* [Architecture Overview](architecture/overview.md)
* [Smart Contracts](architecture/contracts.md)
* [Cross-Chain Bridge](architecture/cross-chain.md)
* [Uniswap v4 Hook](architecture/uniswap-v4-hook.md)
* [Token Model](architecture/token-model.md)
* [Deployed Addresses](deployment/addresses.md)
* [Deployment Guide](deployment/guide.md)
* [DVN Setup](deployment/dvn-setup.md)
* [API Integration](integration/api.md)
* [SDK Usage](integration/sdk.md)

---

## Quick Summary

MultiPools lets anyone launch an ERC-20 token across Ethereum, Base, and Robinhood Chain in a single transaction. The token is immediately tradeable on Uniswap v4 on all three chains with no setup required from the creator.

Key properties:

* Same token address on all three chains (CREATE2 determinism)
* 69 billion total supply split evenly: 23 billion per chain
* 1% swap fee: 70% to creator, 30% to platform
* Liquidity locked permanently from day one
* No bonding curve: pure AMM pricing

---

## Repository Structure

| Repository | Description |
|---|---|
| [contracts](https://github.com/multipools/contracts) | Solidity contracts, tests, and Foundry scripts |
| [launchpad](https://github.com/multipools/launchpad) | Privated |
| [sdk](https://github.com/multipools/sdk) | Privated |
| [scripts](https://github.com/multipools/scripts) | Privated |
| [docs](https://github.com/multipools/docs) | This repository |
