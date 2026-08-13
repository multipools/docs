# Smart Contracts

All contracts are written in Solidity 0.8.26 with `viaIR`, optimizer 200 runs, EVM target Cancun.

---

## MultiPoolsLaunchpadFactory

The main entry point for the protocol. Deployed as a UUPS upgradeable proxy.

Key functions:

* `launch(name, symbol, graffiti, hookSalt, tokenSalt, sqrtPriceX96, tickLower, tickUpper)` — deploys a token, hook, and pool on the source chain, then sends LayerZero messages to remote chains
* `configureTokenDvn(token)` — configures LayerZero ULN settings for a token's DVN
* `setKeeper(keeper)` — sets the authorized keeper address
* `setAdapter(adapter)` — sets the LZ adapter address
* `setSwapHelper(swapHelper)` — sets the swap helper address
* `setTokenContractURI(token, uri)` — sets ERC-7572 contract metadata URI

The factory holds no funds. All fees flow directly to `MultiPoolsFeeVault`.

---

## MultiPoolsToken

ERC-20 token with LayerZero OFT support for native cross-chain transfers.

* Constructor takes only `(name, symbol, graffiti, creator)` so the CREATE2 initcode hash is chain-independent
* `initialize(supply, warpRoute)` injects supply and bridge address post-deploy
* `crosschainMint(to, amount)` is callable only by the trusted bridge on the source chain during remote pool seeding
* Implements ERC-7572 `contractURI()` returning metadata registered via the factory

---

## MultiPoolsHook

Uniswap v4 hook implementing permission bits `0x0ACC` (afterSwap, beforeInitialize, afterInitialize, afterAddLiquidity).

* Charges a fixed 1% fee on every swap, routed to `MultiPoolsFeeVault`
* LP positions added by the seeder are locked forever via a mapping in `afterAddLiquidity`
* `hookSalt` must be mined off-chain so the deployed hook address has the correct lower bits

---

## MultiPoolsFeeVault

Accumulates swap fees per token per creator.

* `accrue(token, creator, amount)` — called by the hook after every swap
* `claimCreator(token)` — transfers 70% of accumulated fees to the creator
* `claimPlatform(token[])` — transfers 30% of accumulated fees to the platform wallet

Fees accumulate in ETH (the swap fee token is always the native token on the chain).

---

## MultiPoolsLZAdapter

OApp adapter implementing the LayerZero V2 messaging interface.

* `send(dstEid, payload, options)` — encodes and sends a cross-chain message
* `_lzReceive(origin, guid, payload, executor, extraData)` — receives a message and triggers remote token deploy via the factory

---

## MultiPoolsDVN

Custom DVN that verifies LayerZero packets for MultiPools routes.

* Six route wallets, each authorized for one directional route
* `verifyAndCommit(srcEid, dstEid, packet, confirmation, nonce)` — signs, verifies, and commits in one atomic transaction
* Uses EIP-191 `signMessage(bytes)` for signatures
* Calls `ReceiveUln302.verify()` then `ReceiveUln302.commitVerification()` on the destination chain

---

## DvnConfigurator

Configures LayerZero ULN DVN settings per token on all three chains.

* `configure(token)` — sets the required DVN list in `SendUln302` and `ReceiveUln302` for a token's OApp
* Called by the keeper after each successful launch

---

## MultiPoolsExecutor

Executes LayerZero packets on the destination chain when automatic execution fails.

* `execute(origin, message)` — calls `lzReceive()` on the destination token contract
* Used as the fallback executor when the standard LZ executor does not trigger

---

## MultiPoolsLiquiditySeeder

Seeds the initial token-only LP position via Uniswap v4 `unlock()`.

* Seeds a position entirely above `tickUpper` so no ETH is required
* The hook locks this position permanently in `afterAddLiquidity`

---

## AutoLPSeeder

Automated seeding bot called by the keeper after a remote token is deployed.

* `triggerAutoLP(token, poolId, currentTick)` — seeds LP if not yet seeded
* Reads `hook.lpSeeded(poolId)` to check seeding status

---

## BuybackBurner

Uses the platform fee share to buy back and burn the launched token.

* `buyback(token, minAmountOut)` — swaps ETH for tokens via `SwapHelper`, then burns them
* Callable only by the platform wallet

---

## RewardsDistributor

Distributes platform fee rewards proportionally to token stakers.

* `distribute(token, amount)` — allocates rewards to stakers
* `claim(token)` — transfers accumulated rewards to the caller

---

## SwapHelper

Executes exact-input swaps via the Uniswap v4 Universal Router.

* Uses negative `amountSpecified` (exact input semantics)
* Applies a configurable slippage tolerance (default 5%)

---

## MultiPoolsTokenLocker

Time-locks tokens on behalf of a user with a configurable release timestamp.

* `lock(token, amount, releaseAt)` — transfers and locks tokens
* `unlock(token)` — releases tokens after the lock expires

---

## ArbVault

Profit-sharing vault for arbitrage proceeds.

* `deposit()` — add ETH to the vault
* `addProfit(amount)` — record profit (onlyOwner)
* `withdraw(amount)` — withdraw principal plus proportional profit share
* 0.1% withdrawal fee
