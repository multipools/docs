# DVN Setup

MultiPools uses a custom Decentralized Verifier Network (DVN) instead of public LayerZero DVNs. This gives full control over verification latency and cost.

---

## Architecture

The DVN consists of two contracts:

| Contract | Address | Purpose |
|---|---|---|
| `MultiPoolsDVN` | `0x829310352947Cc868c910DD133038Bd837EF0000` | Verifies and commits LZ packets |
| `DvnConfigurator` | `0x87BCc0dD2d3e86cffbFF2e4059D2d200C4140000` | Sets ULN config per token post-launch |

Both contracts are deployed at the same address on all three chains (same deployer wallet, same nonce).

---

## Route Wallets

Six dedicated wallets handle six directional routes. Each wallet is registered as `isValidSigner` on the DVN for its destination chain.

| Route | Destination Chain |
|---|---|
| ETH to Base | Base |
| ETH to RBH | Robinhood Chain |
| Base to ETH | Ethereum |
| Base to RBH | Robinhood Chain |
| RBH to ETH | Ethereum |
| RBH to Base | Base |

Using isolated wallets per route eliminates nonce contention when multiple bridge transfers are in flight simultaneously.

---

## Deploy DVN

```bash
forge script script/DeployDvnV2.s.sol \
  --rpc-url $ETH_RPC_URL \
  --broadcast

forge script script/DeployDvnV2.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast

forge script script/DeployDvnV2.s.sol \
  --rpc-url $RBH_RPC_URL \
  --broadcast
```

After deploy, register each route wallet as a valid signer by calling `dvn.addSigner(wallet, dstEid)` on each chain.

---

## Wire DVN into ULN

After deploying, configure the LZ receive library to use the custom DVN for all routes:

```bash
forge script script/DeployDvnCfg.s.sol \
  --rpc-url $ETH_RPC_URL \
  --broadcast

forge script script/DeployDvnCfg.s.sol \
  --rpc-url $BASE_RPC_URL \
  --broadcast

forge script script/DeployDvnCfg.s.sol \
  --rpc-url $RBH_RPC_URL \
  --broadcast
```

This calls `endpoint.setConfig()` with a `UlnConfig` struct pointing to the deployed DVN address.

---

## Per-Token Configuration

Each token launched through the factory needs its DVN config set so the LZ receive library accepts packets verified by the custom DVN. The keeper calls `factory.configureTokenDvn(token)` automatically after each launch.

To manually configure a specific token:

```bash
cast send $FACTORY_ADDR \
  "configureTokenDvn(address)" \
  $TOKEN_ADDR \
  --rpc-url $ETH_RPC_URL \
  --private-key $KEEPER_KEY
```

---

## verifyAndCommit

The DVN keeper process listens for `PacketSent` events emitted by the LZ send library on source chains. When a new packet is detected:

1. Decode the packet header and payload hash.
2. Sign the packet with the route wallet (EIP-191 message signature).
3. Call `dvn.verifyAndCommit(receiveLib, header, payloadHash)` on the destination chain.
4. This calls `ReceiveUln302.verify()` then `commitVerification()` atomically.

After commit, the packet is ready for the executor to deliver.
