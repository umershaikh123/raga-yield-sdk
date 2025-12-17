# Raga Finance - Yield Infrastructure for Neobanks

## Overview

Raga Finance is building a **yield infrastructure layer for Web3 neobanks**, enabling any neobank to instantly offer DeFi investment products to its users across multiple chains and protocols. Through a unified SDK and API suite, neobanks gain access to a standardized interface that abstracts away the complexity of multi-chain smart contracts, yield strategies, data normalization, risk evaluation, and transaction execution.

### Key Principles

- **Protocol Agnostic**: Works with any DeFi protocol
- **Non-Custodial**: Users maintain control of their funds
- **Scalable**: Easy integration of future chains and protocols
- **Cost-Effective**: Minimal infrastructure, optimized for low operational cost over speed

---

## Table of Contents

1. [Product Requirements](#product-requirements)
2. [Tech Stack](#tech-stack)
3. [System Architecture](#system-architecture)
4. [Backend Services](#backend-services)
5. [Data Sources](#data-sources)
6. [Database Design](#database-design)
7. [API Specification](#api-specification)
8. [SDK Structure](#sdk-structure)
9. [Vault System](#vault-system)
10. [Deployment](#deployment)

---

## Product Requirements

### For Neobanks

- Discover high quality yield opportunities
- View historical performance and risk analytics
- Deposit or withdraw into a single vault that manages all strategies
- Add and change strategies according to markets and curators
- See real-time PnL and portfolio performance

### For End Users

- Invest in yield-optimized vaults
- Withdraw money anytime
- See daily PnL of portfolio in dollar value

### Core Functional Requirements

| #   | Requirement           | Description                                                             |
| --- | --------------------- | ----------------------------------------------------------------------- |
| 1   | Protocol Data         | Fetch APY, reward APY, TVL, liquidity, supported networks, risk scores  |
| 2   | Yield Strategies      | Fetch strategies across lending, staking, LPing, restaking, RWA, vaults |
| 3   | Strategy Details      | Retrieve detailed information about specific strategies                 |
| 4   | Transaction Payloads  | Generate deposit and withdraw transaction payloads                      |
| 5   | Transaction Execution | Execute via non-custodial wallets or neobank signing system             |
| 6   | Performance Data      | Historical APY charts, TVL history, volatility indicators               |
| 7   | Risk Analytics        | Risk assessment across strategies                                       |
| 8   | Portfolio Data        | Positions, current value, accrued yield, claimable rewards              |
| 9   | Cross-chain Support   | Handle multi-chain and multi-protocol interactions                      |
| 10  | Data Normalization    | Unified formatting (APY, metadata, decimals, network IDs)               |

### Additional Requirements

| #   | Requirement          | Description                                    |
| --- | -------------------- | ---------------------------------------------- |
| 11  | Market Benchmarks    | Average stablecoin APY, lending averages       |
| 12  | Webhooks             | Callback support for transaction confirmations |
| 13  | Caching              | Improve response times for common data         |
| 14  | Real-time Monitoring | Protocol parameter changes (APY, reward rates) |
| 15  | Error Handling       | Unified errors and simulation endpoints        |
| 16  | Extensibility        | Support future chains/protocols via config     |

### Supported Protocols (v1)

| Protocol | Type               | Description                    |
| -------- | ------------------ | ------------------------------ |
| Aave     | Lending            | Decentralized lending protocol |
| Morpho   | Lending            | Peer-to-peer lending optimizer |
| Pendle   | Yield Tokenization | PT/YT yield trading            |
| Euler    | Lending            | Permissionless lending         |
| Spectra  | Yield Tokenization | Fixed yield protocol           |
| Lido     | Liquid Staking     | ETH liquid staking             |

### Supported Chains (v1)

| Chain    | Chain ID | RPC Required |
| -------- | -------- | ------------ |
| Ethereum | 1        | Yes          |
| Base     | 8453     | Yes          |
| BSC      | 56       | Yes          |
| Arbitrum | 42161    | Yes          |

---

## Tech Stack

### Design Philosophy: Cost Over Speed

This architecture prioritizes **low operational cost** over high performance. We accept slower response times in exchange for minimal infrastructure complexity and cost.

**Trade-offs:**
- Longer data refresh intervals (acceptable for yield data which changes slowly)
- Single-instance services instead of auto-scaling clusters
- In-memory caching instead of distributed cache (initially)
- Consolidated services instead of microservices

### Monorepo Structure

This project uses a **Turborepo monorepo** with pnpm for package management.

```
raga-finance/
├── apps/
│   ├── api/                 # NestJS REST API (consolidated)
│   ├── indexer/             # Ponder indexer
│   └── execution-engine/    # Go fund allocation (TEE)
├── packages/
│   ├── sdk/                 # @raga-finance/sdk
│   ├── types/               # @raga-finance/types
│   ├── contracts/           # ABIs & contract types
│   └── config/              # Shared configs
└── infrastructure/          # Docker, deployment configs
```

### Languages & Frameworks (Simplified)

| Component            | Technology              | Reasoning                                           |
| -------------------- | ----------------------- | --------------------------------------------------- |
| **API Service**      | **NestJS + TypeScript** | Single consolidated service for all backend logic   |
| **Indexer**          | **Ponder**              | TypeScript-based on-chain indexing, replaces Python |
| **Execution Engine** | Go                      | TEE compatibility for secure key management         |
| **SDK**              | TypeScript              | Browser + Node.js compatibility                     |
| **Monorepo**         | Turborepo + pnpm        | Build caching, workspace management                 |

### Why This Stack?

**Single Language (TypeScript):**
- Reduced operational complexity
- Shared types across API, SDK, and indexer
- Easier hiring and maintenance
- No Python runtime/dependencies to manage

**Ponder for Indexing:**
- TypeScript-native, fits our stack
- Built-in database (SQLite/PostgreSQL)
- Handles on-chain event indexing
- Automatic reorg handling
- Lower cost than custom Python scrapers

### Database (Simplified)

| Database   | Purpose                                         | Technology                     |
| ---------- | ----------------------------------------------- | ------------------------------ |
| Primary DB | Users, vaults, strategies, time-series, configs | PostgreSQL (single instance)   |
| Cache      | API response caching                            | In-memory (node-cache) → Redis |

**Note:** We consolidate TimescaleDB into regular PostgreSQL. For time-series queries, we use standard PostgreSQL with proper indexing. This reduces cost and complexity. We can migrate to TimescaleDB later if needed.

### Infrastructure (Minimal)

| Component         | Technology                  | Reasoning                           |
| ----------------- | --------------------------- | ----------------------------------- |
| Hosting           | Railway / Render / Fly.io   | Simple deployment, low cost         |
| Database          | Managed PostgreSQL          | Railway/Render/Supabase             |
| Secrets           | Environment variables       | Platform-native secret management   |
| Monitoring        | Platform built-in + Sentry  | Error tracking without extra infra  |
| CI/CD             | GitHub Actions              | Free for open source                |

**Cost Comparison:**
- Previous (AWS): ~$500-1000/month minimum
- Simplified: ~$50-150/month

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                   CLIENTS                                        │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐                  │
│  │ Neobank Admin   │  │ Neobank App     │  │ End User Wallet │                  │
│  │ Dashboard       │  │ (uses SDK)      │  │                 │                  │
│  └────────┬────────┘  └────────┬────────┘  └────────┬────────┘                  │
└───────────┼─────────────────────┼─────────────────────┼──────────────────────────┘
            │                     │                     │
            ▼                     ▼                     ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         SIMPLIFIED INFRASTRUCTURE                                │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                     API SERVICE (NestJS - Single Instance)                 │  │
│  │                                                                            │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │  │
│  │  │  REST API   │ │  Auth &     │ │  Data       │ │  Vault Deployer     │  │  │
│  │  │  Endpoints  │ │  Rate Limit │ │  Normalizer │ │  (integrated)       │  │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────────────┘  │  │
│  │                                                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     In-Memory Cache (node-cache)                     │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                           │
│  ┌───────────────────────────────────┴───────────────────────────────────────┐  │
│  │                                                                            │  │
│  │  ┌──────────────────────────────┐      ┌────────────────────────────────┐ │  │
│  │  │     PONDER INDEXER           │      │     EXECUTION ENGINE (Go)      │ │  │
│  │  │     (TypeScript)             │      │                                │ │  │
│  │  │                              │      │     - Fund allocation          │ │  │
│  │  │  - On-chain event indexing   │      │     - Withdrawal processing    │ │  │
│  │  │  - Vault deposits/withdraws  │      │     - Rebalancing              │ │  │
│  │  │  - Position tracking         │      │     - TEE secured signing      │ │  │
│  │  │  - Protocol data sync        │      │                                │ │  │
│  │  │  - Historical snapshots      │      │     (Runs separately in TEE)   │ │  │
│  │  └──────────────────────────────┘      └────────────────────────────────┘ │  │
│  │                                                                            │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                           │
│                                      ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                     POSTGRESQL (Single Instance)                           │  │
│  │                                                                            │  │
│  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐  │  │
│  │  │   users     │ │  strategies │ │   vaults    │ │  ponder_* tables    │  │  │
│  │  │   neobanks  │ │  protocols  │ │  positions  │ │  (auto-generated)   │  │  │
│  │  │             │ │  tokens     │ │  txns       │ │                     │  │  │
│  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL DATA SOURCES                                  │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    PROTOCOL APIs (fetched by API service)                │   │
│  │   DeFiLlama │ Morpho GraphQL │ Pendle API │ Lido API │ CoinGecko        │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                    BLOCKCHAIN RPC (used by Ponder)                       │   │
│  │   Ethereum │ Base │ BSC │ Arbitrum (via Alchemy/public RPCs)            │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Backend Services

### 1. API Service (NestJS - Consolidated)

**Purpose**: Single backend service handling all API requests, data fetching, and business logic

**Responsibilities**:

- REST API endpoints (public and authenticated)
- API key authentication and validation
- Rate limiting (in-memory, per API key)
- Protocol data fetching (from external APIs)
- Data normalization (APY/APR standardization)
- Risk score computation
- Vault deployment orchestration
- Webhook dispatch

**Consolidated Components** (previously separate services):
- Data Aggregation (was Python) → TypeScript modules
- Portfolio Tracking → Uses Ponder indexed data
- Vault Deployer Bot → Integrated as a module

**Tech Stack**:

- Framework: NestJS
- Validation: class-validator + class-transformer
- ORM: Prisma
- Cache: node-cache (in-memory)
- API Docs: Swagger (@nestjs/swagger)
- HTTP Client: axios

**Module Structure**:

```
apps/api/src/
├── modules/
│   ├── strategies/          # Strategy discovery & data
│   │   ├── strategies.controller.ts
│   │   ├── strategies.service.ts
│   │   └── fetchers/        # Protocol-specific data fetchers
│   │       ├── aave.fetcher.ts
│   │       ├── morpho.fetcher.ts
│   │       ├── pendle.fetcher.ts
│   │       ├── euler.fetcher.ts
│   │       ├── spectra.fetcher.ts
│   │       ├── lido.fetcher.ts
│   │       └── defillama.fetcher.ts  # Fallback
│   ├── vaults/              # Vault management
│   ├── users/               # User & neobank management
│   ├── portfolio/           # Portfolio (reads from Ponder)
│   ├── transactions/        # Transaction preparation
│   └── deployer/            # Vault deployment
├── common/
│   ├── cache/               # In-memory caching
│   ├── normalizers/         # Data normalization utilities
│   └── guards/              # Auth guards
└── config/
```

### 2. Ponder Indexer (TypeScript)

**Purpose**: Index on-chain events for portfolio tracking and vault metrics

**Why Ponder?**
- TypeScript-native (no Python)
- Built-in SQLite/PostgreSQL support
- Automatic reorg handling
- Type-safe event handlers
- Lower operational complexity

**Responsibilities**:

- Index vault deposit/withdrawal events
- Track user positions on-chain
- Store historical share prices
- Monitor vault TVL changes
- Index protocol-specific events as needed

**What Ponder Indexes**:

```typescript
// ponder.config.ts
export default {
  networks: {
    ethereum: { chainId: 1, rpc: process.env.ETH_RPC_URL },
    base: { chainId: 8453, rpc: process.env.BASE_RPC_URL },
    bsc: { chainId: 56, rpc: process.env.BSC_RPC_URL },
    arbitrum: { chainId: 42161, rpc: process.env.ARB_RPC_URL },
  },
  contracts: {
    RagaVault: {
      abi: RagaVaultABI,
      network: "ethereum",
      address: factory.getDeployedVaults(), // Dynamic
      startBlock: 19000000,
    },
  },
};
```

**Indexed Events**:
- `Deposit(address user, uint256 assets, uint256 shares)`
- `Withdraw(address user, uint256 assets, uint256 shares)`
- `Transfer(address from, address to, uint256 shares)`
- `StrategyAllocationUpdated(...)`

**Ponder Schema**:

```typescript
// ponder.schema.ts
import { createSchema } from "@ponder/core";

export default createSchema((p) => ({
  VaultDeposit: p.createTable({
    id: p.string(),
    vault: p.string(),
    user: p.string(),
    assets: p.bigint(),
    shares: p.bigint(),
    timestamp: p.int(),
    blockNumber: p.int(),
    txHash: p.string(),
  }),

  VaultWithdraw: p.createTable({
    id: p.string(),
    vault: p.string(),
    user: p.string(),
    assets: p.bigint(),
    shares: p.bigint(),
    timestamp: p.int(),
    blockNumber: p.int(),
    txHash: p.string(),
  }),

  UserPosition: p.createTable({
    id: p.string(),           // `${vault}-${user}`
    vault: p.string(),
    user: p.string(),
    shares: p.bigint(),
    totalDeposited: p.bigint(),
    totalWithdrawn: p.bigint(),
    lastUpdated: p.int(),
  }),

  VaultSnapshot: p.createTable({
    id: p.string(),           // `${vault}-${timestamp}`
    vault: p.string(),
    totalAssets: p.bigint(),
    totalShares: p.bigint(),
    sharePrice: p.bigint(),   // Scaled by 1e18
    timestamp: p.int(),
  }),
}));
```

### 3. Execution Engine (Go)

**Purpose**: Execute vault operations securely in TEE

**Responsibilities**:

- Fund allocation to strategies
- Withdrawal processing
- Rebalancing based on curator allocations
- Transaction signing (TEE secured)
- Nonce management
- Gas optimization

**Security**: Runs in Trusted Execution Environment (TEE)

**Note**: This remains in Go for TEE compatibility. It's the only non-TypeScript component.

---

## Data Sources

### Protocol Data Fetching Strategy

Instead of Python scrapers, we fetch protocol data directly in the API service using TypeScript:

| Protocol    | Primary Source   | Fetcher Module         | Update Freq | Caching   |
| ----------- | ---------------- | ---------------------- | ----------- | --------- |
| **Aave**    | Aavescan API     | `aave.fetcher.ts`      | 10 min      | 10 min    |
| **Morpho**  | Morpho GraphQL   | `morpho.fetcher.ts`    | 10 min      | 10 min    |
| **Pendle**  | Pendle V2 API    | `pendle.fetcher.ts`    | 10 min      | 10 min    |
| **Euler**   | Goldsky Subgraph | `euler.fetcher.ts`     | 15 min      | 15 min    |
| **Spectra** | Vision API       | `spectra.fetcher.ts`   | 15 min      | 15 min    |
| **Lido**    | Lido API         | `lido.fetcher.ts`      | 30 min      | 30 min    |
| **Fallback**| DeFiLlama        | `defillama.fetcher.ts` | As needed   | 15 min    |

**Longer update intervals** = lower API costs and simpler caching (acceptable trade-off since yield data doesn't change rapidly)

### On-Chain Data (via Ponder)

| Data Type         | Source              | Update Method      |
| ----------------- | ------------------- | ------------------ |
| Vault Deposits    | Contract events     | Real-time indexing |
| Vault Withdrawals | Contract events     | Real-time indexing |
| User Positions    | Computed from events| Real-time          |
| Share Prices      | Contract calls      | Periodic snapshots |

### RPC Providers (Cost-Optimized)

| Chain    | Provider          | Notes                      |
| -------- | ----------------- | -------------------------- |
| Ethereum | Alchemy (free)    | 300M CU/month free tier    |
| Base     | Alchemy (free)    | Shared quota               |
| BSC      | Public RPC        | Free, reliable enough      |
| Arbitrum | Alchemy (free)    | Shared quota               |

**Note**: Start with free tiers, upgrade only when needed

---

## Database Design

### Simplified Schema (PostgreSQL Only)

We use a single PostgreSQL instance for all data, including time-series. This reduces complexity and cost. For time-series queries, we use standard PostgreSQL with proper indexing and periodic cleanup jobs.

**Note:** Ponder creates its own tables automatically for indexed on-chain data. The schema below is for our application data.

### Prisma Schema

```prisma
// schema.prisma

generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// =====================
// CORE ENTITIES
// =====================

model Neobank {
  id          String   @id @default(cuid())
  name        String
  email       String   @unique
  apiKey      String   @unique @map("api_key")
  apiKeyHash  String   @map("api_key_hash")
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")
  updatedAt   DateTime @updatedAt @map("updated_at")

  users  User[]
  vaults Vault[]

  @@map("neobanks")
}

model User {
  id            String   @id @default(cuid())
  neobankId     String   @map("neobank_id")
  externalId    String?  @map("external_id")
  walletAddress String?  @map("wallet_address")
  email         String?
  name          String?
  isVerified    Boolean  @default(false) @map("is_verified")
  createdAt     DateTime @default(now()) @map("created_at")
  updatedAt     DateTime @updatedAt @map("updated_at")

  neobank      Neobank       @relation(fields: [neobankId], references: [id])
  positions    UserPosition[]
  transactions Transaction[]

  @@unique([neobankId, externalId])
  @@map("users")
}

model Chain {
  id          Int      @id // chain ID
  name        String
  rpcUrl      String   @map("rpc_url")
  explorerUrl String?  @map("explorer_url")
  isActive    Boolean  @default(true) @map("is_active")

  tokens     Token[]
  strategies Strategy[]
  vaults     Vault[]

  @@map("chains")
}

model Protocol {
  id          String   @id @default(cuid())
  name        String
  slug        String   @unique
  logoUrl     String?  @map("logo_url")
  websiteUrl  String?  @map("website_url")
  description String?
  isActive    Boolean  @default(true) @map("is_active")
  createdAt   DateTime @default(now()) @map("created_at")

  strategies Strategy[]

  @@map("protocols")
}

model Token {
  id          String  @id @default(cuid())
  chainId     Int     @map("chain_id")
  address     String
  symbol      String
  name        String?
  decimals    Int
  logoUrl     String? @map("logo_url")
  coingeckoId String? @map("coingecko_id")

  chain      Chain      @relation(fields: [chainId], references: [id])
  strategies Strategy[]

  @@unique([chainId, address])
  @@map("tokens")
}

// =====================
// STRATEGIES
// =====================

model Strategy {
  id              String   @id @default(cuid())
  identifier      String   @unique
  protocolId      String   @map("protocol_id")
  chainId         Int      @map("chain_id")
  name            String
  description     String?
  strategyType    String   @map("strategy_type")
  contractAddress String   @map("contract_address")
  depositTokenId  String?  @map("deposit_token_id")

  // Current metrics (cached from API fetches)
  currentApy      Decimal? @map("current_apy") @db.Decimal(10, 4)
  rewardApy       Decimal? @map("reward_apy") @db.Decimal(10, 4)
  totalApy        Decimal? @map("total_apy") @db.Decimal(10, 4)
  tvlUsd          Decimal? @map("tvl_usd") @db.Decimal(20, 2)
  utilizationRate Decimal? @map("utilization_rate") @db.Decimal(5, 4)

  // Risk
  riskScore   Int?    @map("risk_score")
  riskProfile String? @map("risk_profile")

  isActive    Boolean   @default(true) @map("is_active")
  lastUpdated DateTime? @map("last_updated")
  createdAt   DateTime  @default(now()) @map("created_at")

  protocol     Protocol          @relation(fields: [protocolId], references: [id])
  chain        Chain             @relation(fields: [chainId], references: [id])
  depositToken Token?            @relation(fields: [depositTokenId], references: [id])
  allocations  VaultAllocation[]
  snapshots    StrategySnapshot[]

  @@map("strategies")
}

// =====================
// VAULTS
// =====================

model Vault {
  id           String @id @default(cuid())
  neobankId    String @map("neobank_id")
  chainId      Int    @map("chain_id")
  vaultAddress String @map("vault_address")
  botAddress   String @map("bot_address")

  name        String
  description String?
  vaultType   String? @map("vault_type")

  // Current metrics
  currentApy  Decimal? @map("current_apy") @db.Decimal(10, 4)
  tvlUsd      Decimal? @map("tvl_usd") @db.Decimal(20, 2)
  sharePrice  Decimal? @map("share_price") @db.Decimal(30, 18)
  totalShares Decimal? @map("total_shares") @db.Decimal(30, 18)

  riskProfile String? @map("risk_profile")
  status      String  @default("active")
  isTestnet   Boolean @default(false) @map("is_testnet")

  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  neobank      Neobank           @relation(fields: [neobankId], references: [id])
  chain        Chain             @relation(fields: [chainId], references: [id])
  allocations  VaultAllocation[]
  positions    UserPosition[]
  transactions Transaction[]
  snapshots    VaultSnapshot[]

  @@unique([chainId, vaultAddress])
  @@map("vaults")
}

model VaultAllocation {
  id                   String  @id @default(cuid())
  vaultId              String  @map("vault_id")
  strategyId           String  @map("strategy_id")
  allocationPercentage Decimal @map("allocation_percentage") @db.Decimal(5, 2)
  isActive             Boolean @default(true) @map("is_active")

  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  vault    Vault    @relation(fields: [vaultId], references: [id])
  strategy Strategy @relation(fields: [strategyId], references: [id])

  @@unique([vaultId, strategyId])
  @@map("vault_allocations")
}

// =====================
// USER POSITIONS (synced from Ponder)
// =====================

model UserPosition {
  id      String @id @default(cuid())
  userId  String @map("user_id")
  vaultId String @map("vault_id")

  shares           Decimal  @default(0) @db.Decimal(30, 18)
  costBasisUsd     Decimal  @default(0) @map("cost_basis_usd") @db.Decimal(20, 2)
  totalDepositedUsd Decimal @default(0) @map("total_deposited_usd") @db.Decimal(20, 2)
  totalWithdrawnUsd Decimal @default(0) @map("total_withdrawn_usd") @db.Decimal(20, 2)

  firstDepositAt DateTime? @map("first_deposit_at")
  lastActivityAt DateTime? @map("last_activity_at")

  createdAt DateTime @default(now()) @map("created_at")
  updatedAt DateTime @updatedAt @map("updated_at")

  user  User  @relation(fields: [userId], references: [id])
  vault Vault @relation(fields: [vaultId], references: [id])

  @@unique([userId, vaultId])
  @@map("user_positions")
}

model Transaction {
  id      String @id @default(cuid())
  userId  String @map("user_id")
  vaultId String @map("vault_id")
  chainId Int    @map("chain_id")

  txHash     String @map("tx_hash")
  txType     String @map("tx_type")
  status     String @default("pending")

  amount     Decimal? @db.Decimal(30, 18)
  amountUsd  Decimal? @map("amount_usd") @db.Decimal(20, 2)
  shares     Decimal? @db.Decimal(30, 18)
  sharePrice Decimal? @map("share_price") @db.Decimal(30, 18)

  submittedAt DateTime  @default(now()) @map("submitted_at")
  confirmedAt DateTime? @map("confirmed_at")
  blockNumber BigInt?   @map("block_number")

  createdAt DateTime @default(now()) @map("created_at")

  user  User  @relation(fields: [userId], references: [id])
  vault Vault @relation(fields: [vaultId], references: [id])

  @@index([txHash])
  @@map("transactions")
}

// =====================
// TIME-SERIES (Simple PostgreSQL)
// =====================

model StrategySnapshot {
  id         String   @id @default(cuid())
  strategyId String   @map("strategy_id")
  timestamp  DateTime

  apy        Decimal? @db.Decimal(10, 4)
  rewardApy  Decimal? @map("reward_apy") @db.Decimal(10, 4)
  totalApy   Decimal? @map("total_apy") @db.Decimal(10, 4)
  tvlUsd     Decimal? @map("tvl_usd") @db.Decimal(20, 2)

  strategy Strategy @relation(fields: [strategyId], references: [id])

  @@index([strategyId, timestamp(sort: Desc)])
  @@map("strategy_snapshots")
}

model VaultSnapshot {
  id        String   @id @default(cuid())
  vaultId   String   @map("vault_id")
  timestamp DateTime

  apy         Decimal? @db.Decimal(10, 4)
  tvlUsd      Decimal? @map("tvl_usd") @db.Decimal(20, 2)
  sharePrice  Decimal? @map("share_price") @db.Decimal(30, 18)
  totalShares Decimal? @map("total_shares") @db.Decimal(30, 18)

  vault Vault @relation(fields: [vaultId], references: [id])

  @@index([vaultId, timestamp(sort: Desc)])
  @@map("vault_snapshots")
}

model TokenPrice {
  id        String   @id @default(cuid())
  tokenId   String   @map("token_id")
  timestamp DateTime
  priceUsd  Decimal  @map("price_usd") @db.Decimal(20, 8)

  @@index([tokenId, timestamp(sort: Desc)])
  @@map("token_prices")
}
```

### Data Cleanup (Cron Job)

Since we don't use TimescaleDB retention policies, we run a simple cleanup job:

```typescript
// src/jobs/cleanup.job.ts
@Injectable()
export class CleanupJob {
  constructor(private prisma: PrismaService) {}

  // Run daily via cron
  @Cron('0 3 * * *') // 3 AM daily
  async cleanupOldSnapshots() {
    const ninetyDaysAgo = new Date();
    ninetyDaysAgo.setDate(ninetyDaysAgo.getDate() - 90);

    await this.prisma.strategySnapshot.deleteMany({
      where: { timestamp: { lt: ninetyDaysAgo } },
    });

    await this.prisma.tokenPrice.deleteMany({
      where: { timestamp: { lt: ninetyDaysAgo } },
    });

    // Keep vault snapshots longer (1 year)
    const oneYearAgo = new Date();
    oneYearAgo.setFullYear(oneYearAgo.getFullYear() - 1);

    await this.prisma.vaultSnapshot.deleteMany({
      where: { timestamp: { lt: oneYearAgo } },
    });
  }
}
```

### Ponder Tables (Auto-Generated)

Ponder automatically creates and manages these tables:
- `ponder.VaultDeposit` - All deposit events
- `ponder.VaultWithdraw` - All withdrawal events
- `ponder.UserPosition` - Computed positions
- `ponder.VaultSnapshot` - Periodic vault state

The API service reads from Ponder tables for real-time on-chain data.

---

## API Specification

### Authentication

Two authentication models:

1. **Neobank API Key**: For B2B access (vault management, user portfolio)

   - Header: `X-API-Key: <api_key>`

2. **User Context**: Passed by neobank for user-specific operations
   - Header: `X-User-Id: <user_identifier>`

### Endpoints Overview

#### Public Endpoints (No Auth)

| Method | Endpoint                 | Description                               |
| ------ | ------------------------ | ----------------------------------------- |
| GET    | `/v1/strategies`         | List all available strategies             |
| POST   | `/v1/strategies/details` | Get detailed info for specific strategies |
| GET    | `/v1/protocols`          | List supported protocols                  |
| GET    | `/v1/chains`             | List supported chains                     |
| GET    | `/v1/benchmarks`         | Get market benchmark rates                |

#### Authenticated Endpoints (Neobank API Key)

| Method | Endpoint                 | Description                   |
| ------ | ------------------------ | ----------------------------- |
| POST   | `/v1/vaults`             | Create a new vault            |
| GET    | `/v1/vaults`             | List neobank's vaults         |
| GET    | `/v1/vaults/:id`         | Get vault details             |
| PUT    | `/v1/vaults/:id`         | Update vault allocations      |
| GET    | `/v1/vaults/:id/metrics` | Get vault performance metrics |
| GET    | `/v1/vaults/:id/history` | Get vault historical data     |

#### User Portfolio Endpoints (Neobank API Key + User Context)

| Method | Endpoint                         | Description                    |
| ------ | -------------------------------- | ------------------------------ |
| GET    | `/v1/users/:userId/portfolio`    | Get user's portfolio           |
| GET    | `/v1/users/:userId/positions`    | Get user's positions           |
| GET    | `/v1/users/:userId/transactions` | Get user's transaction history |
| GET    | `/v1/users/:userId/pnl`          | Get user's PnL data            |

#### Transaction Endpoints

| Method | Endpoint                            | Description                  |
| ------ | ----------------------------------- | ---------------------------- |
| POST   | `/v1/transactions/deposit/prepare`  | Prepare deposit transaction  |
| POST   | `/v1/transactions/withdraw/prepare` | Prepare withdraw transaction |
| POST   | `/v1/transactions/simulate`         | Simulate transaction         |
| GET    | `/v1/transactions/:txHash/status`   | Get transaction status       |

### Detailed API Contracts

See [API_SPECIFICATION.md](./API_SPECIFICATION.md) for complete OpenAPI specification.

---

## SDK Structure

### Package Structure

```
@raga-finance/sdk
├── src/
│   ├── index.ts                 # Main exports
│   ├── client/
│   │   ├── RagaClient.ts        # Main SDK client
│   │   ├── HttpClient.ts        # HTTP wrapper
│   │   └── types.ts             # API types
│   ├── modules/
│   │   ├── strategies/          # Strategy discovery
│   │   ├── vaults/              # Vault management
│   │   ├── portfolio/           # Portfolio tracking
│   │   └── transactions/        # Transaction helpers
│   ├── utils/
│   │   ├── formatters.ts        # APY, currency formatters
│   │   ├── validators.ts        # Input validation
│   │   └── constants.ts         # Chain IDs, addresses
│   └── types/
│       ├── strategy.ts
│       ├── vault.ts
│       ├── portfolio.ts
│       └── transaction.ts
├── package.json
├── tsconfig.json
└── README.md
```

### SDK Usage Example

```typescript
import { RagaClient } from "@raga-finance/sdk"

// Initialize client
const raga = new RagaClient({
  apiKey: "your-api-key",
  environment: "mainnet", // or 'testnet'
})

// Fetch all strategies
const strategies = await raga.strategies.getAll({
  chains: [1, 8453], // Ethereum, Base
  types: ["lending", "staking"],
  minApy: 5,
})

// Get strategy details
const details = await raga.strategies.getDetails([
  "strategy-id-1",
  "strategy-id-2",
])

// Create a vault
const vault = await raga.vaults.create({
  chainId: 1,
  name: "Conservative Yield Vault",
  allocations: {
    "aave-usdc-eth": 50,
    "morpho-usdc-eth": 30,
    "lido-eth": 20,
  },
})

// Get user portfolio
const portfolio = await raga.portfolio.get("user-123")

// Prepare deposit transaction
const depositTx = await raga.transactions.prepareDeposit({
  vaultId: vault.id,
  amount: "1000000000", // 1000 USDC (6 decimals)
  userAddress: "0x...",
})

// User signs and broadcasts transaction
// SDK can track status
const status = await raga.transactions.waitForConfirmation(depositTx.txHash)
```

---

## Vault System

### Architecture

Using **Factory Pattern** with Veda's Boring Vault as base:

```
┌─────────────────────────────────────────────────────────────────┐
│                     VAULT FACTORY CONTRACT                       │
│                                                                  │
│  - deployVault(config) → returns vault address                  │
│  - Inherits BoringVault with custom extensions                  │
│  - Registers vault in on-chain registry                         │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
        ┌─────────────────────────────────────────────┐
        │              VAULT INSTANCE                  │
        │                                              │
        │  ┌────────────────────────────────────────┐ │
        │  │         BoringVault (Base)             │ │
        │  │  - ERC4626 compliant                   │ │
        │  │  - Share accounting                    │ │
        │  │  - Deposit/Withdraw logic              │ │
        │  └────────────────────────────────────────┘ │
        │                    │                        │
        │  ┌────────────────────────────────────────┐ │
        │  │      Raga Extensions (Custom)          │ │
        │  │  - Multi-strategy support              │ │
        │  │  - Curator role management             │ │
        │  │  - Fee distribution                    │ │
        │  │  - Emergency controls                  │ │
        │  └────────────────────────────────────────┘ │
        └─────────────────────────────────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            ▼                 ▼                 ▼
     ┌───────────┐     ┌───────────┐     ┌───────────┐
     │ Strategy 1│     │ Strategy 2│     │ Strategy 3│
     │ (Aave)    │     │ (Morpho)  │     │ (Lido)    │
     └───────────┘     └───────────┘     └───────────┘
```

### Vault Deployment Flow

```
1. Neobank calls POST /v1/vaults with configuration
                    │
                    ▼
2. Backend validates request and queues deployment
                    │
                    ▼
3. Vault Deployer Bot picks up job from queue
                    │
                    ▼
4. Bot calls VaultFactory.deployVault() on-chain
                    │
                    ▼
5. Factory deploys new vault instance
                    │
                    ▼
6. Bot configures vault (strategies, allocations, fees)
                    │
                    ▼
7. Bot registers vault in database with status 'active'
                    │
                    ▼
8. Webhook sent to neobank with vault details
```

### Execution Engine Flow

```
┌─────────────────────────────────────────────────────────────────┐
│                     EXECUTION ENGINE (Go)                        │
│                     Running in TEE                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. MONITOR VAULT STATE                                         │
│     - Current allocations vs target allocations                 │
│     - Pending deposits needing allocation                       │
│     - Pending withdrawals needing processing                    │
│                                                                  │
│  2. REBALANCING LOGIC                                           │
│     - Calculate required moves                                  │
│     - Optimize for gas efficiency                               │
│     - Respect slippage limits                                   │
│                                                                  │
│  3. EXECUTION                                                   │
│     - Sign transactions (keys in TEE)                           │
│     - Submit to blockchain                                      │
│     - Monitor confirmation                                      │
│     - Update state                                              │
│                                                                  │
│  4. REPORTING                                                   │
│     - Log all actions                                           │
│     - Update database                                           │
│     - Emit events for monitoring                                │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Deployment

### Simplified Infrastructure (Railway/Render)

We use simple PaaS platforms instead of AWS to minimize cost and operational complexity.

```
┌─────────────────────────────────────────────────────────────────┐
│                     RAILWAY / RENDER                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │                     SERVICES                               │  │
│  │                                                            │  │
│  │  ┌─────────────────┐      ┌─────────────────────────────┐ │  │
│  │  │   API Service   │      │     Ponder Indexer          │ │  │
│  │  │   (NestJS)      │      │     (TypeScript)            │ │  │
│  │  │                 │      │                             │ │  │
│  │  │   - REST API    │      │   - Event indexing          │ │  │
│  │  │   - Cron jobs   │      │   - Position tracking       │ │  │
│  │  │   - Webhooks    │      │   - Snapshots               │ │  │
│  │  │                 │      │                             │ │  │
│  │  │   $20/month     │      │   $20/month                 │ │  │
│  │  └─────────────────┘      └─────────────────────────────┘ │  │
│  │                                                            │  │
│  │  ┌─────────────────────────────────────────────────────┐  │  │
│  │  │              PostgreSQL Database                     │  │  │
│  │  │              (Railway/Supabase)                      │  │  │
│  │  │                                                      │  │  │
│  │  │              $20-50/month                            │  │  │
│  │  └─────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                  │
│  Total: ~$60-100/month (vs $500-1000 on AWS)                    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     SEPARATE HOSTING                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │              Execution Engine (Go)                       │   │
│  │              TEE Provider (Fleek/Phala/Custom)          │   │
│  │                                                          │   │
│  │              - Requires TEE environment                  │   │
│  │              - Separate from main infra                  │   │
│  └─────────────────────────────────────────────────────────┘   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Environment Configuration

| Environment | Purpose           | Platform        | Database               | Cost       |
| ----------- | ----------------- | --------------- | ---------------------- | ---------- |
| Development | Local development | Docker Compose  | PostgreSQL (local)     | $0         |
| Staging     | Testing, QA       | Railway (free)  | Railway PostgreSQL     | $0-20      |
| Production  | Live traffic      | Railway/Render  | Railway/Supabase PG    | $60-100    |

### Recommended Platform: Railway

**Why Railway?**
- Simple GitHub-based deployments
- Native PostgreSQL support
- Environment variable management
- Automatic HTTPS
- Reasonable pricing ($5/service + usage)
- Good DX for small teams

**Alternatives:**
- **Render**: Similar to Railway, slightly cheaper
- **Fly.io**: Better for global distribution
- **Supabase**: Good if you want managed Postgres + extras

### Docker Compose (Development)

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build:
      context: .
      dockerfile: apps/api/Dockerfile
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/raga
      - NODE_ENV=development
    depends_on:
      - db

  indexer:
    build:
      context: .
      dockerfile: apps/indexer/Dockerfile
    environment:
      - DATABASE_URL=postgresql://postgres:postgres@db:5432/raga
      - ETH_RPC_URL=${ETH_RPC_URL}
      - BASE_RPC_URL=${BASE_RPC_URL}
    depends_on:
      - db

  db:
    image: postgres:15-alpine
    environment:
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=postgres
      - POSTGRES_DB=raga
    volumes:
      - postgres_data:/var/lib/postgresql/data
    ports:
      - "5432:5432"

volumes:
  postgres_data:
```

### CI/CD Pipeline (GitHub Actions)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy-api:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: pnpm install

      - name: Run tests
        run: pnpm test

      - name: Deploy to Railway
        uses: railwayapp/railway-action@v1
        with:
          service: api
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}

  deploy-indexer:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Deploy Ponder to Railway
        uses: railwayapp/railway-action@v1
        with:
          service: indexer
        env:
          RAILWAY_TOKEN: ${{ secrets.RAILWAY_TOKEN }}
```

### Environment Variables

```bash
# .env.example

# Database
DATABASE_URL=postgresql://user:password@host:5432/raga

# RPC URLs (use free tiers)
ETH_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/YOUR_KEY
BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/YOUR_KEY
BSC_RPC_URL=https://bsc-dataseed1.binance.org
ARB_RPC_URL=https://arb-mainnet.g.alchemy.com/v2/YOUR_KEY

# API Config
PORT=3000
NODE_ENV=production

# External APIs
DEFILLAMA_API_URL=https://api.llama.fi
COINGECKO_API_URL=https://api.coingecko.com/api/v3

# Monitoring (optional)
SENTRY_DSN=your_sentry_dsn
```

### Scaling Path

When you need to scale beyond the simplified setup:

1. **Add Redis**: When in-memory cache becomes insufficient
2. **Add read replicas**: When database becomes bottleneck
3. **Move to Kubernetes**: When you need auto-scaling
4. **Add CDN**: When you need global distribution

For now, start simple and scale when needed.

---

## Next Steps

1. **Phase 1**: Core Backend Setup
   - [ ] Set up monorepo with Turborepo + pnpm
   - [ ] Initialize NestJS API service
   - [ ] Set up Prisma with PostgreSQL
   - [ ] Implement protocol data fetchers (TypeScript)
   - [ ] Set up in-memory caching

2. **Phase 2**: Ponder Indexer
   - [ ] Set up Ponder project
   - [ ] Define schema for vault events
   - [ ] Implement event handlers
   - [ ] Connect to multi-chain RPCs

3. **Phase 3**: Vault System
   - [ ] Develop vault factory smart contracts
   - [ ] Integrate vault deployment into API
   - [ ] Build execution engine (Go/TEE)

4. **Phase 4**: SDK Development
   - [ ] Design SDK architecture
   - [ ] Implement core modules
   - [ ] Write documentation and examples

5. **Phase 5**: Testing & Launch
   - [ ] Integration testing
   - [ ] Security audit
   - [ ] Testnet deployment (Railway staging)
   - [ ] Production deployment

---

## Architecture Summary

| Previous Architecture | Simplified Architecture |
|-----------------------|------------------------|
| 5 services (API, Python scrapers, Portfolio tracker, Vault bot, Execution engine) | 3 services (API, Ponder indexer, Execution engine) |
| Python + TypeScript + Go | TypeScript + Go only |
| PostgreSQL + TimescaleDB + Redis | PostgreSQL only |
| AWS ECS + RDS + ElastiCache | Railway/Render |
| ~$500-1000/month | ~$60-100/month |

**Key Trade-offs:**
- Slower data refresh (10-15 min vs 5 min) - acceptable for yield data
- Single instance vs auto-scaling - sufficient for early stage
- In-memory cache vs Redis - can add Redis later if needed

---

## References

- [Ponder Documentation](https://ponder.sh/docs)
- [DeFiLlama API](https://defillama.com/docs/api)
- [vaults.fyi Documentation](https://docs.vaults.fyi)
- [Morpho API](https://docs.morpho.org/tools/offchain/api/get-started/)
- [Pendle API](https://docs.pendle.finance/pendle-v2/Developers/Backend/ApiOverview)
- [Euler Subgraphs](https://docs.euler.finance/developers/data-querying/subgraphs/)
- [Lido API](https://docs.lido.fi/integrations/api/)
- [Spectra Developer Docs](https://dev.spectra.finance/)
- [Veda Boring Vault](https://docs.vaults.fyi)
- [Railway Documentation](https://docs.railway.app/)
