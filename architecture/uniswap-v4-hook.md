# Uniswap v4 Hook

MultiPools uses a custom Uniswap v4 hook (`MultiPoolsHook`) to enforce fee collection and permanent LP locking on every pool it creates.

---

## Hook Permission Bits

The hook requires these Uniswap v4 permission bits:

| Permission | Bit | Purpose |
|---|---|---|
| `afterSwap` | enabled | Charge 1% fee on every swap |
| `afterAddLiquidity` | enabled | Track new LP positions |
| `afterRemoveLiquidity` | enabled | Block unauthorized LP withdrawal |

The combined permission flags produce the address suffix `0x0ACC`. Any hook deployed for MultiPools pools must have this suffix in its address. This is enforced by Uniswap v4 at the PoolManager level.

---

## Hook Address Mining

Because the hook address must end in `0x0ACC`, it cannot be deployed at an arbitrary address. A valid CREATE2 salt must be mined off-chain before launch.

The mining process:

1. Compute the hook init code hash for the target factory address (chain-specific).
2. Iterate over candidate salts until `CREATE2(factory, salt, initCodeHash)` produces an address with `0x0ACC` in the lower bits.
3. Pass the valid salt to `launch()`.

The init code hash depends on the factory address, so it is different on each chain. The current values are in [Deployed Addresses](../deployment/addresses.md).

---

## Fee Accrual

`afterSwap()` fires after every swap in a MultiPools pool. It computes the swap fee and calls `FeeVault.accrue()`:

```
swap amount * 100 bps (1%) --> FeeVault.accrue(poolId, creator, amount)
```

Fees accumulate in ETH inside `FeeVault`. The creator's 70% share is immediately claimable. The platform's 30% share is queued for `BuybackBurner` and `RewardsDistributor`.

---

## LP Lock

The hook permanently locks all LP positions in MultiPools pools. It does this by reverting in `afterRemoveLiquidity` for any caller that is not the authorized unlocker address (which is set to the zero address at launch, meaning no one can unlock).

This is a protocol-level guarantee: LP seeded at launch cannot be removed by anyone, including the factory, the keeper, or the creator.

---

## Hook Lock

After the antisnipe window expires (90 seconds after launch), the keeper calls `factory.lockHook(token)`. This sets a `lockForever` flag on the hook, signaling that the hook is now permanently active and cannot be modified.

The `lockForever` flag is cosmetic in terms of on-chain enforcement (the hook already reverts on LP removal), but it serves as a verifiable on-chain signal that the hook configuration is final.
