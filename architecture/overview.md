# Architecture Overview

MultiPools is a multichain token launchpad. One `launch()` call deploys a token on Ethereum, Base, and Robinhood Chain simultaneously, seeds each chain with an AMM pool, and wires cross-chain bridging via a custom LayerZero DVN.

---

## System Components

```
                         User
                          │
                          ▼
             MultiPoolsLaunchpadFactory (UUPS proxy)
                          │
          ┌───────────────┼───────────────┐
          │               │               │
    TokenFactory      HookDeploy      LZAdapter
  (CREATE2 token)   (Uniswap v4)   (OApp message)
          │               │               │
     same addr        pool init      LayerZero V2
    all chains      LP position      │
                    (locked)         ▼
                                MultiPoolsDVN
                                     │
                                     ▼
                              MultiPoolsExecutor
                                     │
                               lzReceive() on
                               remote chain
                                     │
                           repeat TokenFactory +
                           HookDeploy on remote
```

---

## Source Chain Flow

When `launch()` is called on the source chain:

1. **Token deploy** via CREATE2 using `keccak256(abi.encode(creator, name, symbol, nonce))`. This salt produces the same token address on every chain.

2. **Hook deploy** using a pre-mined hook salt. The hook address must have Uniswap v4 permission bits `0x0ACC` set in the lower bytes.

3. **Pool initialization** via Uniswap v4 `PoolManager.initialize()`. The pool is configured with the hook and initial `sqrtPriceX96`.

4. **Liquidity seeding** via `MultiPoolsLiquiditySeeder`. The token-only position is seeded above `tickUpper` so no ETH is required from the launcher. The position is locked forever by the hook.

5. **LayerZero message** sent via `MultiPoolsLZAdapter` to each remote chain carrying: `tokenSalt`, `hookSalt`, `creator`, `sqrtPriceX96`, `tickLower`, `tickUpper`.

---

## Remote Chain Flow

On each remote chain, the LayerZero message triggers:

1. **MultiPoolsDVN** signs the packet with the route-specific wallet, calls `ReceiveUln302.verify()`, then calls `commitVerification()` in one atomic transaction.

2. **MultiPoolsExecutor** calls `lzReceive()` on the destination chain, which repeats steps 1 through 4 of the source chain flow using the same salts from the message.

---

## DVN Architecture

MultiPools runs a custom DVN (Decentralized Verifier Network) instead of relying on public LayerZero DVNs.

Key design decisions:

* Six isolated route wallets, one per directional LZ route (ETH→Base, ETH→RBH, Base→ETH, Base→RBH, RBH→ETH, RBH→Base)
* Each wallet is registered as `isValidSigner` on the DVN contract for its destination chain
* `verifyAndCommit()` is atomic: one transaction signs, verifies, and commits
* No shared signer to eliminate nonce contention under burst traffic

---

## Fee Flow

```
afterSwap() fires on every swap
        │
        └── swap amount × 100 bps (1%) → FeeVault.accrue(token, creator)
                                                    │
                                        ┌───────────┴──────────────┐
                                        │                          │
                                   70% creator             30% platform
                                        │                          │
                                 claimCreator()          claimPlatform()
                                        │                          │
                                   ETH transfer           BuybackBurner
                                                               │
                                                 buy back token + burn
                                                 OR distribute via
                                                 RewardsDistributor
```

---

## Contract Addresses

All contracts are deployed at the same address on Ethereum (chain ID 1), Base (chain ID 8453), and Robinhood Chain (chain ID 4663).

| Contract | Address |
|---|---|
| Factory Proxy (UUPS) | `0x61A4e6e6ceCfc04f44719C15F88D1d3Eea270000` |
| Token Factory | `0xd47D7B2CA130e58F175A83d9B4C53E06BD425489` |
| LZ Adapter | `0x0fc53A7E94d0d92df70462c91819e808f127FD01` |
| Fee Vault | `0x5d8B6AD5C9EA8D3136dAC5c114bFd5C2Ff44cbc9` |
| Liquidity Seeder | `0xA811d26a6ec52F1f3b636FE11D1e2c266626bb5e` |
| Auto LP Seeder | `0xe3caD21cccD2fB9D4D42F464Fe99599C08D03973` |
| Buyback Burner | `0xe8e7456F17De4d805F6A6C831da93eF075560E53` |
| Rewards Distributor | `0x042671f5B6DC64389f955bcb8D31A3Cb24A54864` |
| Swap Helper | `0x02360B08Ac321Bea472d404C77033B1bfCfD7134` |
| MultiPools DVN | `0x829310352947Cc868c910DD133038Bd837EF0000` |
| DVN Configurator | `0x87BCc0dD2d3e86cffbFF2e4059D2d200C4140000` |
