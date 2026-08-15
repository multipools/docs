# API Integration

The MultiPools REST API provides read access to token data, bridge status, fee vault state, and arbitrage history. All endpoints return JSON.

Base URL: `https://multipools.trade/api`

---

## Tokens

### List Tokens

```
GET /api/tokens?page=1&limit=20
```

Query parameters:

| Parameter | Type | Description |
|---|---|---|
| `page` | integer | Page number, starting from 1 |
| `limit` | integer | Results per page, max 100 |
| `creator` | address | Filter by creator wallet |
| `search` | string | Search by name or symbol |

Response:

```json
{
  "tokens": [
    {
      "id": "...",
      "name": "My Token",
      "symbol": "MTK",
      "tokenAddress": "0x...",
      "creatorAddress": "0x...",
      "status": "live",
      "createdAt": "2026-08-01T00:00:00.000Z",
      "marketCap": "125000.00",
      "volume24h": "8400.00",
      "priceUsd": "0.000181"
    }
  ],
  "total": 61,
  "page": 1,
  "limit": 20
}
```

### Get Token

```
GET /api/tokens/:address
```

Returns full token data including hook address, pool ID, and vault balances.

### Get Token Metadata (ERC-7572)

```
GET /api/tokens/:address/metadata
```

Returns ERC-7572-compatible JSON metadata for the token contract.

---

## Bridge

### Track Bridge Transfer

```
POST /api/bridge/track
```

Register a bridge transfer for status tracking. The keeper calls this automatically when it detects a `PacketSent` event.

### Get Bridge Status

```
GET /api/bridge/status?srcTxHash=0x...
```

Returns the current status of a bridge transfer.

| Status | Description |
|---|---|
| `pending` | DVN has not yet verified the packet |
| `delivered` | lzReceive() executed successfully on destination |
| `failed` | Delivery failed after retries |

---

## Fee Vault

### Get Vault Balances

```
GET /api/vault/balances?tokenAddress=0x...
```

Returns claimable creator and platform fee balances for a given token.

```json
{
  "tokenAddress": "0x...",
  "creatorClaimable": "0.042",
  "platformClaimable": "0.018",
  "totalVolumeEth": "4.2"
}
```

---

## Statistics

### Protocol Stats

```
GET /api/stats
```

```json
{
  "totalTokens": 61,
  "totalVolumeEth": "892.4",
  "totalFeeEth": "8.924"
}
```

---

## Arbitrage

### Get Current Prices

```
GET /api/arb/prices
```

Returns current token prices on each chain for arbitrage detection.

### Get Arbitrage History

```
GET /api/arb/history
```

Returns recent arbitrage executions with chain, token, and profit data.

---

## Locks

### Get Token Locks

```
GET /api/locks?tokenAddress=0x...
```

Returns active LP lock records for a token.

---

## Authentication

All read endpoints are public. Write endpoints (bridge tracking, keeper operations) require a keeper API secret passed via the `X-Keeper-Secret` header.
