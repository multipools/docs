# Cross-Chain Bridge

MultiPools uses LayerZero V2 for all cross-chain messaging and token transfers.

---

## LayerZero Integration

MultiPools implements the LayerZero OApp (Omni-Application) pattern via `MultiPoolsLZAdapter`. Each token is its own OApp peer on every chain.

Peers are set after a token is launched:

```
ETH token.setPeer(BASE_EID, base_token_address)
ETH token.setPeer(RBH_EID, rbh_token_address)
Base token.setPeer(ETH_EID, eth_token_address)
Base token.setPeer(RBH_EID, rbh_token_address)
RBH token.setPeer(ETH_EID, eth_token_address)
RBH token.setPeer(BASE_EID, base_token_address)
```

Since token addresses are deterministic (CREATE2), all peers resolve to the same address.

---

## Custom DVN

MultiPools operates a custom DVN instead of relying on public LayerZero DVNs.

### Why a Custom DVN

Public DVNs add latency and cost. The MultiPools DVN is optimized for the three-chain route set, processes packets immediately after they appear on-chain, and uses per-route wallets to avoid nonce contention.

### DVN Architecture

```
Source chain emits PacketSent event
        │
        ▼
DVN worker detects event
        │
        ▼
verifyAndCommit(srcEid, dstEid, packet, ...)
        │
        ├── sign(keccak256(packet)) with route wallet (EIP-191)
        ├── ReceiveUln302.verify(packet, signatures, confirmations)
        └── ReceiveUln302.commitVerification(packet, confirmations)
```

### Route Wallets

| Route | Description |
|---|---|
| RBH → ETH | Signs packets leaving Robinhood Chain bound for Ethereum |
| Base → ETH | Signs packets leaving Base bound for Ethereum |
| ETH → Base | Signs packets leaving Ethereum bound for Base |
| RBH → Base | Signs packets leaving Robinhood Chain bound for Base |
| ETH → RBH | Signs packets leaving Ethereum bound for Robinhood Chain |
| Base → RBH | Signs packets leaving Base bound for Robinhood Chain |

Each wallet is registered as `isValidSigner` on `MultiPoolsDVN` for its destination chain only.

---

## Self-Executor

`MultiPoolsExecutor` provides a fallback execution path when the standard LayerZero executor does not call `lzReceive()` on the destination chain.

The keeper monitors `PacketSent` events and calls `execute()` on the executor after verification is confirmed.

---

## Token Bridging (OFT)

Once a token is launched on all chains, holders can bridge tokens using the OFT interface:

```ts
// Bridge 100 tokens from Ethereum to Base
await token.send({
  dstEid: BASE_EID,
  to: recipientAddress,
  amountLD: parseEther("100"),
  minAmountLD: parseEther("95"),
  extraOptions: "0x00030100110100000000000000000000000000030D40",
  composeMsg: "0x",
  oftCmd: "0x",
}, { nativeFee, lzTokenFee: 0n }, sender)
```

LZ options encode 200,000 gas for the destination `lzReceive()` call.

---

## Failure Recovery

If a cross-chain launch fails partway through:

1. The keeper detects the stuck state by scanning `launch_deliveries` in the database
2. On startup and every 60 seconds, the keeper retries any pending deliveries
3. If `lzReceive()` has been called but pool seeding failed, `AutoLPSeeder.triggerAutoLP()` is called to complete seeding

---

## Chain IDs and Endpoints

| Chain | Chain ID | LayerZero EID | LayerZero Endpoint |
|---|---|---|---|
| Ethereum | 1 | 30101 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Base | 8453 | 30184 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Robinhood Chain | 4663 | 30416 | `0x6F475642a6e85809B1c36Fa62763669b1b48DD5B` |
