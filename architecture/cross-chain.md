# Cross-Chain Bridge

MultiPools uses LayerZero V2 for all cross-chain communication. Tokens are native OFTs, meaning the same contract instance exists on each chain at the same address. There is no lock-and-mint wrapping.

---

## Token Bridging

After a token is launched and all three chains have seeded pools, users can bridge tokens between chains using the OFT `send()` function directly on the token contract.

```
User calls token.send(dstEid, recipient, amount, fee)
        |
        v
LZ EndpointV2.send() on source chain
        |
        v
MultiPoolsDVN signs and verifies on destination chain
        |
        v
MultiPoolsExecutor calls lzReceive() on destination chain
        |
        v
token.lzReceive() mints tokens on destination
```

Bridging requires a LayerZero messaging fee paid in ETH on the source chain. The fee can be quoted with `token.quoteSend(dstEid, ...)`.

---

## Launch Cross-Chain Delivery

On launch, the factory sends two LZ messages (one to each remote chain) via `MultiPoolsLZAdapter`. These messages carry everything the remote chain needs to deploy the token, hook, and pool without any further user interaction.

Message payload:

| Field | Type | Description |
|---|---|---|
| `tokenSalt` | `bytes32` | CREATE2 salt for token deploy (same on all chains) |
| `hookSalt` | `bytes32` | CREATE2 salt for hook deploy (chain-specific, pre-mined) |
| `creator` | `address` | Creator wallet address |
| `sqrtPriceX96` | `uint160` | Initial pool price |
| `tickLower` | `int24` | Lower tick of LP position |
| `tickUpper` | `int24` | Upper tick of LP position |

---

## DVN: Verification Flow

MultiPools runs its own DVN instead of using public LayerZero DVNs. This gives full control over verification latency, cost, and security.

### Route Wallets

Six wallets handle six directional routes:

| Route | Responsibility |
|---|---|
| ETH to Base | Signs packets originating on Ethereum destined for Base |
| ETH to RBH | Signs packets originating on Ethereum destined for Robinhood Chain |
| Base to ETH | Signs packets originating on Base destined for Ethereum |
| Base to RBH | Signs packets originating on Base destined for Robinhood Chain |
| RBH to ETH | Signs packets originating on Robinhood Chain destined for Ethereum |
| RBH to Base | Signs packets originating on Robinhood Chain destined for Base |

Each wallet is registered as `isValidSigner` on the DVN contract for its destination chain only. This means a compromised route wallet can only affect one directional route.

### verifyAndCommit

The DVN calls one function atomically:

```solidity
function verifyAndCommit(
    address receiveLib,
    PacketHeader calldata header,
    bytes32 payloadHash
) external;
```

This calls:
1. `ReceiveUln302.verify(header, payloadHash, confirmations)`
2. `ReceiveUln302.commitVerification(header, payloadHash)`

In one transaction. This eliminates the two-step verify + commit gap where a packet could be verified but not yet executable.

---

## Executor: Packet Delivery

After the DVN commits verification, the packet is ready for execution. `MultiPoolsExecutor` polls `inboundPayloadHash()` on the destination chain to detect when verification is committed, then calls `lzReceive()`.

```solidity
function executePacket(packet) {
    // poll until inboundPayloadHash() returns non-zero
    // then call endpoint.lzReceive(origin, receiver, guid, message, extraData)
}
```

Polling interval is approximately 3 seconds. Execution typically completes within 30-60 seconds of the source chain transaction being included.

---

## LZ Endpoint Addresses

| Chain | Chain ID | EID | Endpoint Address |
|---|---|---|---|
| Ethereum | 1 | 30101 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Base | 8453 | 30184 | `0x1a44076050125825900e736c501f859c50fE728c` |
| Robinhood Chain | 4663 | 30416 | `0x6F475642a6e85809B1c36Fa62763669b1b48DD5B` |

Note: Robinhood Chain uses a different endpoint address from the canonical LayerZero endpoint used on Ethereum and Base.
