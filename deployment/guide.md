# Deployment Guide

This guide covers deploying the full MultiPools protocol stack to a new chain or redeploying after an upgrade.

See [Deployed Addresses](addresses.md) for current production addresses.

---

## Prerequisites

- [Foundry](https://getfoundry.sh) installed
- Node.js 18+ and pnpm
- RPC endpoints for all three chains
- Deployer wallet with ETH on all chains

---

## Environment Variables

Create a `.env` file (never commit this):

```
DEPLOYER_PRIVATE_KEY=0x...
ALCHEMY_API_KEY=...
ETHERSCAN_API_KEY=...
BASESCAN_API_KEY=...

# RPC
ETH_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/YOUR_KEY
RBH_RPC_URL=https://rpc.mainnet.chain.robinhood.com
```

---

## Build

```bash
git clone --recurse-submodules https://github.com/multipools/contracts
cd contracts
npm install
forge build
```

---

## Deploy All Contracts

The `DeployV21.s.sol` script deploys all core contracts in one run per chain.

```bash
# Ethereum
forge script script/DeployV21.s.sol \
  --rpc-url $ETH_RPC_URL \
  --broadcast \
  --verify \
  -vvvv

# Base
forge script script/DeployV21.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast \
  --verify \
  -vvvv

# Robinhood Chain (uses Blockscout/Sourcify, not Etherscan)
forge script script/DeployV21.s.sol \
  --rpc-url $RBH_RPC_URL \
  --broadcast \
  -vvvv
```

For Robinhood Chain contract verification, use Sourcify:

```bash
forge verify-contract \
  --verifier sourcify \
  --verifier-url https://sourcify.dev/server/v2 \
  --chain 4663 \
  CONTRACT_ADDRESS \
  src/MultiPoolsLaunchpadFactory.sol:MultiPoolsLaunchpadFactory
```

---

## Wire LayerZero Peers

After deploying on all three chains, set LZ peers so the adapter contracts trust each other:

```bash
forge script script/WireV21.s.sol \
  --rpc-url $ETH_RPC_URL \
  --broadcast

forge script script/WireV21.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast

forge script script/WireV21.s.sol \
  --rpc-url $RBH_RPC_URL \
  --broadcast
```

---

## Compute Hook Init Hash

After deploying a new factory, compute the hook init hash for each chain. This value is used off-chain to mine valid hook salts:

```bash
forge script script/ComputeHookInitHash.s.sol --rpc-url $ETH_RPC_URL
forge script script/ComputeHookInitHash.s.sol --rpc-url $BASE_RPC_URL
forge script script/ComputeHookInitHash.s.sol --rpc-url $RBH_RPC_URL
```

Update the keeper environment variables with the new hashes.

---

## Deploy DVN

See [DVN Setup](dvn-setup.md) for full DVN deployment instructions.

---

## Upgrade Factory

The factory is UUPS upgradeable. To upgrade:

```bash
forge script script/UpgradeFactory.s.sol \
  --rpc-url $ETH_RPC_URL \
  --broadcast
```

The script deploys a new implementation and calls `upgradeToAndCall()` on the proxy.
