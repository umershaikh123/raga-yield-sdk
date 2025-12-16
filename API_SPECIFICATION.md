# Raga Finance - API Specification

## Overview

This document defines the complete REST API specification for the Raga Finance yield infrastructure platform.

**Base URL**: `https://api.raga.finance`

**API Version**: `v1`

---

## Table of Contents

1. [Authentication](#authentication)
2. [Common Response Formats](#common-response-formats)
3. [Error Handling](#error-handling)
4. [Rate Limiting](#rate-limiting)
5. [Public Endpoints](#public-endpoints)
6. [Neobank Endpoints](#neobank-endpoints)
7. [User Portfolio Endpoints](#user-portfolio-endpoints)
8. [Transaction Endpoints](#transaction-endpoints)
9. [Webhook Events](#webhook-events)

---

## Authentication

### API Key Authentication

All authenticated endpoints require an API key passed in the header:

```
X-API-Key: your-api-key-here
```

### User Context

For user-specific operations, pass the user identifier:

```
X-User-Id: user-external-id
X-User-Address: 0x... (optional, for wallet-based identification)
```

### Example Request

```bash
curl -X GET "https://api.raga.finance/v1/vaults" \
  -H "X-API-Key: raga_live_abc123..." \
  -H "Content-Type: application/json"
```

---

## Common Response Formats

### Success Response

```json
{
  "success": true,
  "data": { ... },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

### Paginated Response

```json
{
  "success": true,
  "data": [ ... ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 150,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

---

## Error Handling

### Error Response Format

```json
{
  "success": false,
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Invalid chain_id provided",
    "details": {
      "field": "chain_id",
      "value": 999,
      "allowedValues": [1, 8453, 56, 42161]
    }
  },
  "meta": {
    "timestamp": "2024-01-15T10:30:00Z",
    "requestId": "req_abc123"
  }
}
```

### Error Codes

| Code | HTTP Status | Description |
|------|-------------|-------------|
| `UNAUTHORIZED` | 401 | Invalid or missing API key |
| `FORBIDDEN` | 403 | Insufficient permissions |
| `NOT_FOUND` | 404 | Resource not found |
| `VALIDATION_ERROR` | 400 | Invalid request parameters |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests |
| `INTERNAL_ERROR` | 500 | Internal server error |
| `SERVICE_UNAVAILABLE` | 503 | Service temporarily unavailable |

---

## Rate Limiting

| Tier | Requests/Minute | Requests/Hour |
|------|-----------------|---------------|
| Free | 60 | 1,000 |
| Standard | 300 | 10,000 |
| Enterprise | 1,000 | 50,000 |

Rate limit headers included in responses:

```
X-RateLimit-Limit: 300
X-RateLimit-Remaining: 295
X-RateLimit-Reset: 1705312200
```

---

## Public Endpoints

### GET /v1/strategies

Retrieve all available yield strategies.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chain_id` | integer | No | Filter by chain ID |
| `chain_ids` | string | No | Comma-separated chain IDs |
| `protocol` | string | No | Filter by protocol slug |
| `type` | string | No | Strategy type (lending, staking, lp, vault) |
| `min_apy` | number | No | Minimum APY filter |
| `max_apy` | number | No | Maximum APY filter |
| `min_tvl` | number | No | Minimum TVL in USD |
| `risk_profile` | string | No | low, medium, high |
| `sort_by` | string | No | apy, tvl, risk_score (default: tvl) |
| `sort_order` | string | No | asc, desc (default: desc) |
| `page` | integer | No | Page number (default: 1) |
| `page_size` | integer | No | Items per page (default: 20, max: 100) |

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "strat_aave_usdc_eth",
      "identifier": "aave-v3-usdc-ethereum",
      "name": "Aave V3 USDC",
      "protocol": {
        "id": "aave",
        "name": "Aave",
        "logo": "https://assets.raga.finance/protocols/aave.svg"
      },
      "chain": {
        "id": 1,
        "name": "Ethereum",
        "logo": "https://assets.raga.finance/chains/ethereum.svg"
      },
      "type": "lending",
      "depositToken": {
        "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "symbol": "USDC",
        "decimals": 6,
        "logo": "https://assets.raga.finance/tokens/usdc.svg"
      },
      "contractAddress": "0x...",
      "metrics": {
        "apy": 4.52,
        "rewardApy": 0.85,
        "totalApy": 5.37,
        "tvlUsd": 1250000000,
        "utilizationRate": 0.82
      },
      "risk": {
        "score": 2,
        "profile": "low",
        "factors": ["audited", "battle-tested", "high-liquidity"]
      },
      "isActive": true,
      "updatedAt": "2024-01-15T10:25:00Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 156,
    "totalPages": 8,
    "hasNext": true,
    "hasPrev": false
  }
}
```

---

### POST /v1/strategies/details

Get detailed information for specific strategies.

**Request Body:**

```json
{
  "identifiers": [
    "aave-v3-usdc-ethereum",
    "morpho-usdc-ethereum",
    "lido-eth-ethereum"
  ]
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "aave-v3-usdc-ethereum": {
      "id": "strat_aave_usdc_eth",
      "identifier": "aave-v3-usdc-ethereum",
      "name": "Aave V3 USDC",
      "description": "Supply USDC to Aave V3 on Ethereum mainnet to earn interest from borrowers.",
      "protocol": {
        "id": "aave",
        "name": "Aave",
        "logo": "https://assets.raga.finance/protocols/aave.svg",
        "website": "https://aave.com"
      },
      "chain": {
        "id": 1,
        "name": "Ethereum"
      },
      "type": "lending",
      "depositToken": {
        "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "symbol": "USDC",
        "name": "USD Coin",
        "decimals": 6
      },
      "receiptToken": {
        "address": "0x...",
        "symbol": "aUSDC",
        "decimals": 6
      },
      "contractAddress": "0x87870Bca3F3fD6335C3F4ce8392D69350B4fA4E2",
      "metrics": {
        "apy": 4.52,
        "rewardApy": 0.85,
        "totalApy": 5.37,
        "apy7d": 4.48,
        "apy30d": 4.65,
        "tvlUsd": 1250000000,
        "utilizationRate": 0.82,
        "borrowApy": 6.25,
        "supplyApy": 4.52
      },
      "risk": {
        "score": 2,
        "profile": "low",
        "factors": [
          {
            "name": "Smart Contract Risk",
            "level": "low",
            "description": "Multiple audits, bug bounty program"
          },
          {
            "name": "Liquidity Risk",
            "level": "low",
            "description": "High liquidity, instant withdrawals"
          }
        ],
        "audits": [
          {
            "auditor": "OpenZeppelin",
            "date": "2023-06-15",
            "reportUrl": "https://..."
          }
        ]
      },
      "charts": {
        "apy": {
          "1704067200": 4.25,
          "1704153600": 4.35,
          "1704240000": 4.52
        },
        "tvl": {
          "1704067200": 1200000000,
          "1704153600": 1225000000,
          "1704240000": 1250000000
        }
      },
      "isActive": true,
      "updatedAt": "2024-01-15T10:25:00Z"
    },
    "morpho-usdc-ethereum": {
      // ... similar structure
    }
  }
}
```

---

### GET /v1/protocols

List all supported protocols.

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "aave",
      "name": "Aave",
      "slug": "aave",
      "logo": "https://assets.raga.finance/protocols/aave.svg",
      "website": "https://aave.com",
      "description": "Decentralized non-custodial liquidity protocol",
      "chains": [1, 8453, 42161],
      "strategyTypes": ["lending"],
      "totalTvlUsd": 15000000000,
      "strategyCount": 45,
      "isActive": true
    },
    {
      "id": "morpho",
      "name": "Morpho",
      "slug": "morpho",
      "logo": "https://assets.raga.finance/protocols/morpho.svg",
      "website": "https://morpho.org",
      "description": "Peer-to-peer lending optimizer",
      "chains": [1, 8453],
      "strategyTypes": ["lending"],
      "totalTvlUsd": 3000000000,
      "strategyCount": 28,
      "isActive": true
    }
  ]
}
```

---

### GET /v1/chains

List supported chains.

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": 1,
      "name": "Ethereum",
      "slug": "ethereum",
      "logo": "https://assets.raga.finance/chains/ethereum.svg",
      "explorerUrl": "https://etherscan.io",
      "nativeCurrency": {
        "symbol": "ETH",
        "decimals": 18
      },
      "isActive": true
    },
    {
      "id": 8453,
      "name": "Base",
      "slug": "base",
      "logo": "https://assets.raga.finance/chains/base.svg",
      "explorerUrl": "https://basescan.org",
      "nativeCurrency": {
        "symbol": "ETH",
        "decimals": 18
      },
      "isActive": true
    }
  ]
}
```

---

### GET /v1/benchmarks

Get market benchmark rates.

**Response:**

```json
{
  "success": true,
  "data": {
    "stablecoin": {
      "avgApy": 4.85,
      "medianApy": 4.52,
      "minApy": 2.15,
      "maxApy": 12.50,
      "protocols": {
        "aave": 4.52,
        "morpho": 5.25,
        "compound": 3.85
      }
    },
    "eth": {
      "stakingApy": 3.45,
      "lsdApy": 3.65,
      "protocols": {
        "lido": 3.45,
        "rocketpool": 3.52
      }
    },
    "lending": {
      "avgSupplyApy": 4.25,
      "avgBorrowApy": 6.85
    },
    "updatedAt": "2024-01-15T10:00:00Z"
  }
}
```

---

## Neobank Endpoints

*Requires API Key Authentication*

### POST /v1/neobanks/register

Register a new neobank account.

**Request Body:**

```json
{
  "name": "NeoBank Inc",
  "email": "admin@neobank.com",
  "website": "https://neobank.com",
  "contactName": "John Doe",
  "contactPhone": "+1234567890"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "nb_abc123",
    "name": "NeoBank Inc",
    "email": "admin@neobank.com",
    "apiKey": "raga_live_sk_abc123...",
    "createdAt": "2024-01-15T10:30:00Z"
  }
}
```

---

### POST /v1/vaults

Create a new vault for the neobank.

**Request Body:**

```json
{
  "chainId": 1,
  "name": "Conservative USD Vault",
  "description": "Low-risk stablecoin yield vault",
  "depositToken": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
  "allocations": {
    "aave-v3-usdc-ethereum": 50,
    "morpho-usdc-ethereum": 30,
    "compound-v3-usdc-ethereum": 20
  },
  "feePercentage": 0.5,
  "isTestnet": false
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "vault_xyz789",
    "status": "deploying",
    "chainId": 1,
    "name": "Conservative USD Vault",
    "description": "Low-risk stablecoin yield vault",
    "depositToken": {
      "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "symbol": "USDC",
      "decimals": 6
    },
    "allocations": [
      {
        "strategyId": "aave-v3-usdc-ethereum",
        "percentage": 50
      },
      {
        "strategyId": "morpho-usdc-ethereum",
        "percentage": 30
      },
      {
        "strategyId": "compound-v3-usdc-ethereum",
        "percentage": 20
      }
    ],
    "feePercentage": 0.5,
    "isTestnet": false,
    "createdAt": "2024-01-15T10:30:00Z",
    "estimatedDeploymentTime": "2024-01-15T10:35:00Z"
  }
}
```

---

### GET /v1/vaults

List all vaults for the neobank.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `chain_id` | integer | No | Filter by chain |
| `status` | string | No | deploying, active, paused |
| `is_testnet` | boolean | No | Filter testnet/mainnet |

**Response:**

```json
{
  "success": true,
  "data": {
    "testnet": [
      {
        "id": "vault_test123",
        "vaultAddress": "0x1234567890123456789012345678901234567890",
        "botAddress": "0x0987654321098765432109876543210987654321",
        "chainId": 11155111,
        "name": "Test Vault",
        "status": "active",
        "isActive": true,
        "metrics": {
          "apy": 5.25,
          "tvlUsd": 100000,
          "sharePrice": "1.05"
        },
        "riskProfile": "low",
        "charts": {
          "apy": { "...": "..." },
          "tvl": { "...": "..." }
        },
        "createdAt": "2024-01-10T08:00:00Z"
      }
    ],
    "mainnet": [
      {
        "id": "vault_xyz789",
        "vaultAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
        "botAddress": "0xfedcba0987654321fedcba0987654321fedcba09",
        "chainId": 1,
        "name": "Conservative USD Vault",
        "status": "active",
        "isActive": true,
        "metrics": {
          "apy": 5.37,
          "tvlUsd": 5250000,
          "sharePrice": "1.0285"
        },
        "riskProfile": "low",
        "allocations": [
          {
            "strategyId": "aave-v3-usdc-ethereum",
            "strategyName": "Aave V3 USDC",
            "percentage": 50,
            "currentValue": 2625000
          }
        ],
        "charts": {
          "apy": {
            "1704067200": 5.15,
            "1704153600": 5.28,
            "1704240000": 5.37
          },
          "tvl": {
            "1704067200": 5000000,
            "1704153600": 5125000,
            "1704240000": 5250000
          }
        },
        "createdAt": "2024-01-01T00:00:00Z"
      }
    ]
  }
}
```

---

### GET /v1/vaults/:vaultId

Get detailed vault information.

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "vault_xyz789",
    "vaultAddress": "0xabcdef1234567890abcdef1234567890abcdef12",
    "botAddress": "0xfedcba0987654321fedcba0987654321fedcba09",
    "chainId": 1,
    "chain": {
      "id": 1,
      "name": "Ethereum",
      "explorerUrl": "https://etherscan.io"
    },
    "name": "Conservative USD Vault",
    "description": "Low-risk stablecoin yield vault",
    "status": "active",
    "depositToken": {
      "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
      "symbol": "USDC",
      "decimals": 6
    },
    "shareToken": {
      "address": "0xabcdef1234567890abcdef1234567890abcdef12",
      "symbol": "rvUSDC",
      "decimals": 6
    },
    "metrics": {
      "apy": 5.37,
      "apy7d": 5.28,
      "apy30d": 5.15,
      "tvlUsd": 5250000,
      "totalShares": "5102184523152",
      "sharePrice": "1.028945",
      "totalDeposited": 5500000,
      "totalWithdrawn": 350000,
      "totalYieldGenerated": 100000
    },
    "allocations": [
      {
        "strategyId": "aave-v3-usdc-ethereum",
        "strategyName": "Aave V3 USDC",
        "protocol": "Aave",
        "targetPercentage": 50,
        "currentPercentage": 49.5,
        "currentValue": 2598750,
        "apy": 4.52
      },
      {
        "strategyId": "morpho-usdc-ethereum",
        "strategyName": "Morpho USDC",
        "protocol": "Morpho",
        "targetPercentage": 30,
        "currentPercentage": 30.2,
        "currentValue": 1585500,
        "apy": 5.85
      },
      {
        "strategyId": "compound-v3-usdc-ethereum",
        "strategyName": "Compound V3 USDC",
        "protocol": "Compound",
        "targetPercentage": 20,
        "currentPercentage": 20.3,
        "currentValue": 1065750,
        "apy": 3.95
      }
    ],
    "riskProfile": "low",
    "feePercentage": 0.5,
    "feeVaultAddress": "0x...",
    "charts": {
      "apy": { "...": "..." },
      "tvl": { "...": "..." },
      "sharePrice": { "...": "..." }
    },
    "recentTransactions": [
      {
        "txHash": "0x...",
        "type": "deposit",
        "amount": 10000,
        "timestamp": "2024-01-15T09:00:00Z"
      }
    ],
    "isTestnet": false,
    "createdAt": "2024-01-01T00:00:00Z",
    "updatedAt": "2024-01-15T10:30:00Z"
  }
}
```

---

### PUT /v1/vaults/:vaultId

Update vault configuration (allocations).

**Request Body:**

```json
{
  "allocations": {
    "aave-v3-usdc-ethereum": 40,
    "morpho-usdc-ethereum": 40,
    "compound-v3-usdc-ethereum": 20
  }
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "id": "vault_xyz789",
    "status": "rebalancing",
    "allocations": [
      {
        "strategyId": "aave-v3-usdc-ethereum",
        "targetPercentage": 40,
        "currentPercentage": 50
      },
      {
        "strategyId": "morpho-usdc-ethereum",
        "targetPercentage": 40,
        "currentPercentage": 30
      },
      {
        "strategyId": "compound-v3-usdc-ethereum",
        "targetPercentage": 20,
        "currentPercentage": 20
      }
    ],
    "rebalanceEstimate": {
      "estimatedGas": "0.05 ETH",
      "estimatedTime": "2024-01-15T10:45:00Z"
    },
    "updatedAt": "2024-01-15T10:35:00Z"
  }
}
```

---

## User Portfolio Endpoints

*Requires API Key + User Context*

### GET /v1/users/:userId/portfolio

Get user's complete portfolio across all vaults.

**Response:**

```json
{
  "success": true,
  "data": {
    "userId": "user_123",
    "walletAddress": "0x1234567890123456789012345678901234567890",
    "summary": {
      "totalValueUsd": 25750.50,
      "totalDepositedUsd": 25000.00,
      "totalPnlUsd": 750.50,
      "totalPnlPercentage": 3.00,
      "avgApy": 5.15
    },
    "positions": [
      {
        "vaultId": "vault_xyz789",
        "vaultName": "Conservative USD Vault",
        "vaultAddress": "0xabcdef...",
        "chainId": 1,
        "depositToken": {
          "symbol": "USDC",
          "address": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48"
        },
        "shares": "10000000000",
        "sharePrice": "1.028945",
        "currentValueUsd": 10289.45,
        "depositedValueUsd": 10000.00,
        "pnlUsd": 289.45,
        "pnlPercentage": 2.89,
        "avgBuyPrice": "1.0",
        "firstDepositAt": "2024-01-01T10:00:00Z",
        "lastActivityAt": "2024-01-10T15:30:00Z"
      }
    ],
    "charts": {
      "portfolioValue": {
        "1704067200": 24500,
        "1704153600": 25125,
        "1704240000": 25750.50
      },
      "pnl": {
        "1704067200": 200,
        "1704153600": 475,
        "1704240000": 750.50
      }
    }
  }
}
```

---

### GET /v1/users/:userId/positions

Get user's positions in a specific vault.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `vault_id` | string | No | Filter by vault |

**Response:**

```json
{
  "success": true,
  "data": {
    "positions": [
      {
        "vaultId": "vault_xyz789",
        "shares": "10000000000",
        "sharePrice": "1.028945",
        "currentValueUsd": 10289.45,
        "costBasisUsd": 10000.00,
        "unrealizedPnlUsd": 289.45,
        "unrealizedPnlPercentage": 2.89,
        "avgEntryPrice": "1.0",
        "deposits": [
          {
            "txHash": "0x...",
            "amount": "5000000000",
            "amountUsd": 5000,
            "sharePrice": "1.0",
            "timestamp": "2024-01-01T10:00:00Z"
          },
          {
            "txHash": "0x...",
            "amount": "5000000000",
            "amountUsd": 5000,
            "sharePrice": "1.0",
            "timestamp": "2024-01-05T14:00:00Z"
          }
        ]
      }
    ]
  }
}
```

---

### GET /v1/users/:userId/transactions

Get user's transaction history.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `vault_id` | string | No | Filter by vault |
| `type` | string | No | deposit, withdraw, claim |
| `status` | string | No | pending, confirmed, failed |
| `from_date` | string | No | ISO date string |
| `to_date` | string | No | ISO date string |
| `page` | integer | No | Page number |
| `page_size` | integer | No | Items per page |

**Response:**

```json
{
  "success": true,
  "data": [
    {
      "id": "tx_abc123",
      "txHash": "0x1234...",
      "vaultId": "vault_xyz789",
      "vaultName": "Conservative USD Vault",
      "chainId": 1,
      "type": "deposit",
      "status": "confirmed",
      "amount": "5000000000",
      "amountUsd": 5000.00,
      "shares": "4855282352",
      "sharePrice": "1.0298",
      "gasPaid": "0.0025 ETH",
      "submittedAt": "2024-01-10T15:28:00Z",
      "confirmedAt": "2024-01-10T15:30:00Z",
      "blockNumber": 19012345
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "totalItems": 5,
    "totalPages": 1
  }
}
```

---

### GET /v1/users/:userId/pnl

Get user's PnL breakdown.

**Query Parameters:**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `period` | string | No | 1d, 7d, 30d, 90d, 1y, all (default: 30d) |
| `vault_id` | string | No | Filter by vault |

**Response:**

```json
{
  "success": true,
  "data": {
    "period": "30d",
    "summary": {
      "startValueUsd": 24000.00,
      "endValueUsd": 25750.50,
      "netDepositsUsd": 1000.00,
      "realizedPnlUsd": 150.00,
      "unrealizedPnlUsd": 600.50,
      "totalPnlUsd": 750.50,
      "totalPnlPercentage": 3.00,
      "yieldEarnedUsd": 750.50
    },
    "byVault": [
      {
        "vaultId": "vault_xyz789",
        "vaultName": "Conservative USD Vault",
        "startValueUsd": 14000.00,
        "endValueUsd": 15500.00,
        "pnlUsd": 500.00,
        "pnlPercentage": 3.33,
        "apy": 5.37
      }
    ],
    "dailyPnl": {
      "2024-01-14": { "value": 25500, "pnl": 25.50 },
      "2024-01-15": { "value": 25750.50, "pnl": 28.25 }
    }
  }
}
```

---

## Transaction Endpoints

### POST /v1/transactions/deposit/prepare

Prepare a deposit transaction payload.

**Request Body:**

```json
{
  "vaultId": "vault_xyz789",
  "userAddress": "0x1234567890123456789012345678901234567890",
  "amount": "1000000000",
  "slippageTolerance": 0.5
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "transactionId": "prep_tx_123",
    "chainId": 1,
    "steps": [
      {
        "step": 1,
        "type": "approval",
        "description": "Approve USDC spending",
        "to": "0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48",
        "data": "0x095ea7b3...",
        "value": "0",
        "gasEstimate": "46000"
      },
      {
        "step": 2,
        "type": "deposit",
        "description": "Deposit into vault",
        "to": "0xabcdef1234567890abcdef1234567890abcdef12",
        "data": "0x6e553f65...",
        "value": "0",
        "gasEstimate": "150000"
      }
    ],
    "summary": {
      "depositAmount": "1000000000",
      "depositAmountFormatted": "1000 USDC",
      "estimatedShares": "972176523",
      "estimatedSharePrice": "1.0286",
      "minSharesReceived": "967315640",
      "totalGasEstimate": "196000",
      "estimatedGasCostUsd": 15.50
    },
    "expiresAt": "2024-01-15T10:45:00Z"
  }
}
```

---

### POST /v1/transactions/withdraw/prepare

Prepare a withdrawal transaction payload.

**Request Body:**

```json
{
  "vaultId": "vault_xyz789",
  "userAddress": "0x1234567890123456789012345678901234567890",
  "shares": "500000000",
  "slippageTolerance": 0.5
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "transactionId": "prep_tx_456",
    "chainId": 1,
    "steps": [
      {
        "step": 1,
        "type": "withdraw",
        "description": "Withdraw from vault",
        "to": "0xabcdef1234567890abcdef1234567890abcdef12",
        "data": "0x2e1a7d4d...",
        "value": "0",
        "gasEstimate": "200000"
      }
    ],
    "summary": {
      "sharesRedeemed": "500000000",
      "estimatedReceive": "514472500",
      "estimatedReceiveFormatted": "514.47 USDC",
      "minReceive": "511900137",
      "sharePrice": "1.0289",
      "estimatedGasCostUsd": 12.00
    },
    "expiresAt": "2024-01-15T10:45:00Z"
  }
}
```

---

### POST /v1/transactions/simulate

Simulate a transaction before execution.

**Request Body:**

```json
{
  "chainId": 1,
  "from": "0x1234567890123456789012345678901234567890",
  "to": "0xabcdef1234567890abcdef1234567890abcdef12",
  "data": "0x6e553f65...",
  "value": "0"
}
```

**Response:**

```json
{
  "success": true,
  "data": {
    "success": true,
    "gasUsed": "145235",
    "gasLimit": "196000",
    "returnData": "0x...",
    "logs": [
      {
        "address": "0x...",
        "topics": ["0x..."],
        "data": "0x..."
      }
    ],
    "stateChanges": [
      {
        "address": "0x...",
        "balanceBefore": "1000000000",
        "balanceAfter": "0"
      }
    ]
  }
}
```

---

### GET /v1/transactions/:txHash/status

Get transaction status.

**Response:**

```json
{
  "success": true,
  "data": {
    "txHash": "0x1234...",
    "chainId": 1,
    "status": "confirmed",
    "type": "deposit",
    "blockNumber": 19012345,
    "blockTimestamp": "2024-01-15T10:30:00Z",
    "confirmations": 12,
    "gasUsed": "145235",
    "effectiveGasPrice": "25000000000",
    "gasCostEth": "0.00363",
    "gasCostUsd": 8.50,
    "events": [
      {
        "name": "Deposit",
        "args": {
          "sender": "0x...",
          "owner": "0x...",
          "assets": "1000000000",
          "shares": "972176523"
        }
      }
    ]
  }
}
```

---

## Webhook Events

### Webhook Configuration

Configure webhooks via dashboard or API:

```json
{
  "url": "https://your-app.com/webhooks/raga",
  "secret": "whsec_abc123...",
  "events": [
    "vault.deployed",
    "vault.rebalanced",
    "transaction.confirmed",
    "transaction.failed"
  ]
}
```

### Event Types

| Event | Description |
|-------|-------------|
| `vault.deployed` | Vault deployment completed |
| `vault.rebalanced` | Vault rebalancing completed |
| `vault.paused` | Vault paused |
| `vault.resumed` | Vault resumed |
| `transaction.pending` | Transaction submitted |
| `transaction.confirmed` | Transaction confirmed |
| `transaction.failed` | Transaction failed |
| `apy.updated` | Significant APY change |

### Webhook Payload

```json
{
  "id": "evt_abc123",
  "type": "vault.deployed",
  "timestamp": "2024-01-15T10:35:00Z",
  "data": {
    "vaultId": "vault_xyz789",
    "vaultAddress": "0xabcdef...",
    "chainId": 1,
    "status": "active"
  }
}
```

### Webhook Signature Verification

```typescript
import crypto from 'crypto';

function verifyWebhook(payload: string, signature: string, secret: string): boolean {
  const expectedSignature = crypto
    .createHmac('sha256', secret)
    .update(payload)
    .digest('hex');

  return crypto.timingSafeEqual(
    Buffer.from(signature),
    Buffer.from(`sha256=${expectedSignature}`)
  );
}
```

---

## SDK Quick Reference

```typescript
import { RagaClient } from '@raga-finance/sdk';

const raga = new RagaClient({ apiKey: 'your-api-key' });

// Strategies
raga.strategies.getAll(filters)
raga.strategies.getDetails(identifiers)

// Vaults
raga.vaults.create(config)
raga.vaults.list(filters)
raga.vaults.get(vaultId)
raga.vaults.update(vaultId, allocations)

// Portfolio
raga.portfolio.get(userId)
raga.portfolio.getPositions(userId, vaultId)
raga.portfolio.getTransactions(userId, filters)
raga.portfolio.getPnl(userId, period)

// Transactions
raga.transactions.prepareDeposit(params)
raga.transactions.prepareWithdraw(params)
raga.transactions.simulate(tx)
raga.transactions.getStatus(txHash)

// Utilities
raga.protocols.list()
raga.chains.list()
raga.benchmarks.get()
```

---

## Changelog

### v1.0.0 (2024-01-15)
- Initial API release
- Support for Aave, Morpho, Pendle, Euler, Spectra, Lido
- Ethereum, Base, BSC, Arbitrum chains
