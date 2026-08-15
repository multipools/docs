# Smart Contracts

Reference for every contract in the MultiPools protocol.

---

## MultiPoolsLaunchpadFactory

**Address:** `0x61A4e6e6ceCfc04f44719C15F88D1d3Eea270000` (all chains, UUPS proxy)

Entry point for the launch flow. Accepts `launch()` calls, orchestrates token deploy, hook deploy, pool initialization, LP seeding, and cross-chain LZ message. Upgradeable via UUPS pattern with keeper as upgrade authority.

Key functions:

| Function | Description |
|---|---|
| `launch(params)` | Deploy token, init pool, seed LP, send LZ message to remote chains |
| `configureTokenDvn(token)` | Set DVN ULN config for a given token (called by keeper post-launch) |
| `setTokenContractURI(token, uri)` | Set ERC-7572 metadata URI on a deployed token |
| `lockHook(token)` | Lock the hook permanently (called after antisnipe window expires) |
| `renounceTokenOwnership(token)` | Renounce ownership of the token contract |

---

## MultiPoolsToken

ERC-20 token deployed via CREATE2. Each token launched through the factory is an instance of this contract. Implements LayerZero OFT for native cross-chain transfers without wrapping.

Key properties:

| Property | Value |
|---|---|
| Total supply | 69,000,000,000 (69 billion) |
| Standard | ERC-20 plus LayerZero OFT |
| Max wallet | 2% of supply for the first 60 seconds after launch |
| Metadata | ERC-7572 `contractURI()` |

The same CREATE2 salt produces the same token address on all three chains. The salt is `keccak256(abi.encode(creator, name, symbol, nonce))` where nonce is mined off-chain to produce an address with no conflicting hook bits.

---

## MultiPoolsHook

Uniswap v4 hook with permission flags `0x0ACC` (afterSwap, afterAddLiquidity, afterRemoveLiquidity).

Key behaviors:

- Charges 1% fee on every swap, accrued to `MultiPoolsFeeVault`
- Blocks LP withdrawal by reverting in `afterRemoveLiquidity` for non-authorized callers
- Tracks per-pool fee accumulation for creator and platform splits
- Sets `lockForever` flag after antisnipe window to signal that the hook is permanently active

Hook address is deterministic via CREATE2. The salt must be mined off-chain so the hook address has the correct Uniswap v4 permission bits. The init hash is factory-specific and chain-specific.

---

## MultiPoolsFeeVault

Accumulates swap fees per token per creator.

Fee split:

| Share | Amount | Recipient |
|---|---|---|
| Creator | 70% | Directly claimable by creator wallet |
| Platform | 30% | Routed to BuybackBurner and RewardsDistributor |

Functions:

| Function | Description |
|---|---|
| `claimCreator(token)` | Creator claims their accumulated ETH fees |
| `claimPlatform(token)` | Platform claims its accumulated ETH fees |

---

## MultiPoolsLZAdapter

OFTAdapter that sends cross-chain OApp messages. Called by the factory on launch to notify remote chains.

Message payload includes: `tokenSalt`, `hookSalt`, `creator`, `sqrtPriceX96`, `tickLower`, `tickUpper`.

Uses LayerZero V2 `endpoint.send()` with the factory as the OApp owner. Peers are set to the adapter address on each remote chain.

---

## MultiPoolsDVN

Custom DVN. Verifies and commits LZ packets on the destination chain.

Architecture:

- Six isolated route wallets (one per directional route)
- Each wallet is `isValidSigner` on the DVN for its destination chain
- `verifyAndCommit()` calls `ReceiveUln302.verify()` then `commitVerification()` atomically
- Registered in ULN config via `DvnConfigurator` per token

**Address:** `0x829310352947Cc868c910DD133038Bd837EF0000` (all chains)

---

## MultiPoolsExecutor

Calls `lzReceive()` on destination chains after DVN verification is committed. Polls `inboundPayloadHash()` to detect when the packet is ready, then executes.

**Address:** `0x8673aFB1196d09F09957F25BFBC939152b4E0000` (all chains)

---

## MultiPoolsLiquiditySeeder

Seeds token-only LP positions into Uniswap v4 pools via `PoolManager.modifyLiquidity()`. Positions are placed above `tickUpper` so they require only tokens (no ETH). The hook prevents these positions from being withdrawn.

---

## AutoLPSeeder

Accumulates platform fee ETH then auto-compounds it back into LP positions. Called by the keeper periodically.

---

## BuybackBurner

Receives platform fee ETH, uses it to buy the launched token via Uniswap v4, then burns the purchased tokens. Reduces circulating supply over time.

---

## RewardsDistributor

Distributes token rewards from platform fees to LP holders. Tracks LP share proportionally.

---

## SwapHelper

Executes exact-input swaps against Uniswap v4 via `PoolManager.unlock()` / `unlockCallback`. Used internally by BuybackBurner and by the frontend for price quotes.

---

## ArbVault

Profit-sharing vault for arbitrage operations. Accepts ETH deposits, tracks shares, and adds profit via `addProfit()` (owner only). Charges a 0.1% withdrawal fee.

---

## MultiPoolsKeeperProxy

Keeper-owned proxy for permissioned protocol operations like `setPeer()` on LZ adapter, `configureTokenDvn()`, and factory upgrades. Allows the keeper wallet to act on behalf of the protocol owner for specific functions.

---

## MultiPoolsTokenLocker

Locks token positions for a configurable period. Used during the antisnipe window to prevent early LP withdrawal.

---

## DvnConfigLib

Library encoding `UlnConfig` structs for `IMessageLibManager.setConfig()`. The struct must be ABI-encoded as a full struct with a leading 32-byte offset, not field-by-field.

---

## DvnConfigurator

Sets DVN ULN config per token on all three chains so `MultiPoolsDVN` is recognized as the required verifier by the LZ receive library. Called by the keeper after each token launch.
