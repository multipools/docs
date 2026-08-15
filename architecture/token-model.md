# Token Model

Every token launched through MultiPools follows the same supply distribution and omnichain design.

---

## Supply

| Property | Value |
|---|---|
| Total supply | 69,000,000,000 (69 billion) |
| Decimals | 18 |
| Standard | ERC-20 plus LayerZero OFT |

---

## Distribution

Supply is split across three chains at launch. There is no presale, no team allocation, and no vesting.

| Chain | Amount | Share |
|---|---|---|
| Source chain (launch chain) | 42,000,000,000 | ~60.9% |
| Remote chain 1 | 13,500,000,000 | ~19.6% |
| Remote chain 2 | 13,500,000,000 | ~19.6% |

All tokens are seeded directly into Uniswap v4 pools at launch. No tokens go to the creator, the platform, or any other wallet.

---

## Omnichain Design

Each token is a NativeOFT: the same contract instance exists on Ethereum, Base, and Robinhood Chain at the same address. Tokens are not wrapped or locked when bridging between chains. The OFT standard burns on the source chain and mints on the destination chain.

The same CREATE2 address across all chains is achieved by using the same salt (`keccak256(abi.encode(creator, name, symbol, nonce))`) and the same `TokenFactory` address (deployed at the same address on all chains from the same deployer wallet at the same nonce).

---

## Anti-Snipe Protection

For the first 60 seconds after launch:

- Maximum wallet size: 2% of total supply (1,380,000,000 tokens per chain)
- Any transfer that would put the recipient over the limit is rejected

After 60 seconds, the max wallet restriction is lifted automatically. No transaction or call is needed to remove it.

---

## Metadata

Tokens implement ERC-7572 `contractURI()`. The metadata URI is set by the factory keeper after launch and points to a JSON endpoint that returns token name, symbol, description, image, and social links.

---

## Ownership

Token ownership is renounced shortly after launch. The keeper calls `factory.renounceTokenOwnership(token)` after the hook lock step. After renouncement, no one can call owner-only functions on the token (including setting new peers or changing the contract URI).

---

## Bridging

Users bridge tokens using the standard OFT interface:

```solidity
// Quote the fee
(MessagingFee memory fee, ) = token.quoteSend(SendParam({
    dstEid: 30184,         // Base EID
    to: bytes32(uint256(uint160(recipient))),
    amountLD: amount,
    minAmountLD: amount,
    extraOptions: "",
    composeMsg: "",
    oftCmd: ""
}), false);

// Send
token.send{value: fee.nativeFee}(sendParam, fee, refundTo);
```

The bridge completes in approximately 30 to 60 seconds end-to-end (source confirm + DVN verify + executor deliver).
