# Architecture Overview

MultiPools is an omnichain token launchpad. One `launch()` call deploys a token on Ethereum, Base, and Robinhood Chain simultaneously, seeds each chain with a Uniswap v4 AMM pool, and wires cross-chain bridging via a custom LayerZero DVN.

---

## System Components

```
                         User
                          |
                          v
             MultiPoolsLaunchpadFactory (UUPS proxy)
                          |
          +---------------+---------------+
          |               |               |
    TokenFactory      HookDeploy      LZAdapter
  (CREATE2 token)   (Uniswap v4)   (OApp message)
          |               |               |
     same addr        pool init      LayerZero V2
    all chains      LP position          |
                    (locked)             v
                                  MultiPoolsDVN
                                        |
                                        v
                                 MultiPoolsExecutor
                                        |
                                  lzReceive() on
                                  remote chain
                                        |
                              repeat TokenFactory +
                              HookDeploy on remote
```

---

## Source Chain Flow

When `launch()` is called on the source chain:

1. **Token deploy** via CREATE2 using `keccak256(abi.encode(creator, name, symbol, nonce))`. This salt produces the same token address on every chain.

2. **Hook deploy** using a pre-mined hook salt. The hook address must have Uniswap v4 permission bits `0x0ACC` set in the lower bytes.

3. **Pool initialization** via Uniswap v4 `PoolManager.initialize()`. The pool is configured with the hook and initial `sqrtPriceX96`.

4. **Liquidity seeding** via `MultiPoolsLiquiditySeeder`. A token-only position is seeded above `tickUpper` so no ETH is required from the launcher. The position is locked forever by the hook.

5. **LayerZero message** sent via `MultiPoolsLZAdapter` to each remote chain carrying: `tokenSalt`, `hookSalt`, `creator`, `sqrtPriceX96`, `tickLower`, `tickUpper`.

---

## Remote Chain Flow

On each remote chain, the LayerZero message triggers:

1. **MultiPoolsDVN** signs the packet with the route-specific wallet, calls `ReceiveUln302.verify()`, then calls `commitVerification()` in one atomic transaction.

2. **MultiPoolsExecutor** calls `lzReceive()` on the destination chain, which repeats steps 1 through 4 of the source chain flow using the same salts from the message.

---

## DVN Architecture

MultiPools runs a custom DVN instead of relying on public LayerZero DVNs.

Key design decisions:

- Six isolated route wallets, one per directional LZ route (ETH-Base, ETH-RBH, Base-ETH, Base-RBH, RBH-ETH, RBH-Base)
- Each wallet is registered as `isValidSigner` on the DVN contract for its destination chain
- `verifyAndCommit()` is atomic: one transaction signs, verifies, and commits
- No shared signer, eliminating nonce contention under burst traffic

---

## Fee Flow

```
afterSwap() fires on every swap
        |
        v
FeeVault.accrue(token, creator, amount)
        |
        +-- 70% --> creator (claimCreator())
        |
        +-- 30% --> platform
                        |
                        +-- BuybackBurner: buys token and burns
                        |
                        +-- RewardsDistributor: distributes to LP holders
```
