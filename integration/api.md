# API Integration

The MultiPools API server exposes a REST API for querying token data, bridge history, price feeds, and arbitrage state.

Base URL: `https://multipools.trade/api`

---

## Endpoints

### Tokens

#### List All Tokens

```
GET /api/tokens
```

Returns all launched tokens with market data.

Response:

```json
[
  {
    "id": "uuid",
    "tokenAddress": "0x...",
    "name": "My Token",
    "symbol": "MYT",
    "creatorAddress": "0x...",
    "chainId": 1,
    "hookAddress": "0x...",
    "launchedAt": "2026-08-13T00:00:00Z",
    "priceUsd": "0.0001",
    "volume24h": "50000",
    "marketCap": "6900000"
  }
]
```

#### Get Token by Address

```
GET /api/tokens/:address
```

Returns full token details including pool configuration and fee stats.

#### Get Token Trades

```
GET /api/tokens/:address/trades
```

Returns recent swap events for a token.

#### Get Token Holders

```
GET /api/tokens/:address/holders
```

Returns current holder list and balances.

#### Get On-Chain Price

```
GET /api/tokens/:address/price
```

Returns the current price from Uniswap v4 via `extsload` on the PoolManager slot.

#### Get Contract Metadata (ERC-7572)

```
GET /api/tokens/:address/metadata
```

Returns ERC-7572 compliant contract metadata for the token.

---

### Bridge

#### List Bridge Transactions

```
GET /api/bridge
```

Returns recent cross-chain bridge deliveries with status.

Response:

```json
[
  {
    "id": "uuid",
    "tokenAddress": "0x...",
    "srcChainId": 1,
    "dstChainId": 8453,
    "txHash": "0x...",
    "status": "delivered",
    "createdAt": "2026-08-13T00:00:00Z"
  }
]
```

---

### Pre-Seed

#### Submit Pre-Seed Request

```
POST /api/preseed
Content-Type: application/json

{
  "tokenAddress": "0x...",
  "chainId": 1,
  "creator": "0x..."
}
```

Signals the keeper to prepare LP seeding parameters for an upcoming launch.

---

### Arbitrage

#### Get Price Feed

```
GET /api/arb/prices
```

Returns current token prices across all three chains.

#### Get Arbitrage History

```
GET /api/arb/history
```

Returns recent arbitrage executions with profit data.

---

### Locks

#### Get Token Locks

```
GET /api/locks?tokenAddress=0x...
```

Returns active token lock records.

---

## Authentication

The API is public for read operations. Write operations (pre-seed, admin) require a session token obtained via the platform authentication flow.

---

## TypeScript SDK

Instead of calling the REST API directly, use the TypeScript SDK for typed access with React Query integration:

```bash
pnpm add @multipools/api-client
```

See the [SDK documentation](sdk.md) for usage examples.
