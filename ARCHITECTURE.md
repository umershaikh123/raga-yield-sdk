# Raga Finance - Yield Infrastructure for Neobanks

## Overview

Raga Finance is building a **yield infrastructure layer for Web3 neobanks**, enabling any neobank to instantly offer DeFi investment products to its users across multiple chains and protocols. Through a unified SDK and API suite, neobanks gain access to a standardized interface that abstracts away the complexity of multi-chain smart contracts, yield strategies, data normalization, risk evaluation, and transaction execution.

### Key Principles

- **Protocol Agnostic**: Works with any DeFi protocol
- **Non-Custodial**: Users maintain control of their funds
- **Scalable**: Easy integration of future chains and protocols

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

### Monorepo Structure

This project uses a **Turborepo monorepo** with pnpm for package management.

For detailed project structure, see: [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md)

For design principles and best practices, see: [DESIGN_PRINCIPLES.md](./DESIGN_PRINCIPLES.md)

### Languages & Frameworks

| Component            | Technology              | Reasoning                                                      |
| -------------------- | ----------------------- | -------------------------------------------------------------- |
| **API Service**      | **NestJS + TypeScript** | Enterprise-grade, DI, microservices-ready, built-in validation |
| **Data Workers**     | Python                  | Data processing, protocol adapters                             |
| **Execution Engine** | Go                      | Performance critical, TEE compatibility                        |
| **SDK**              | TypeScript              | Browser + Node.js compatibility                                |
| **Monorepo**         | Turborepo + pnpm        | Build caching, workspace management                            |

### Why NestJS?

- **Enterprise Architecture**: Modular, opinionated structure ideal for complex business logic
- **Dependency Injection**: Built-in DI container for testability and maintainability
- **Microservices Support**: Native support for Redis, Kafka, RabbitMQ, gRPC
- **TypeScript First**: Full TypeScript support with decorators
- **Built-in Features**: Validation, caching, logging, testing utilities
- **Swagger**: Automatic API documentation generation

Reference: [Node.js Frameworks Comparison 2024](https://dev.to/encore/nodejs-frameworks-roundup-2024-elysia-hono-nest-encore-which-should-you-pick-19oj)

### Databases

| Database       | Purpose                                    | Technology              |
| -------------- | ------------------------------------------ | ----------------------- |
| Primary DB     | Users, vaults, strategies, configs         | PostgreSQL (AWS RDS)    |
| Time-Series DB | APY/TVL history, price data, PnL snapshots | TimescaleDB (AWS RDS)   |
| Cache          | Hot data, rate limiting, sessions          | Redis (AWS ElastiCache) |
| Message Queue  | Async job processing                       | Redis Streams           |

### Infrastructure

| Component               | Technology                  | Reasoning                     |
| ----------------------- | --------------------------- | ----------------------------- |
| Container Orchestration | AWS ECS + Fargate           | Simpler than K8s, pay-per-use |
| Load Balancer           | AWS ALB                     | Native ECS integration        |
| API Gateway             | AWS API Gateway (optional)  | Rate limiting, API keys       |
| Secrets                 | AWS Secrets Manager         | Secure credential storage     |
| Monitoring              | AWS CloudWatch + Prometheus | Metrics and alerting          |
| CI/CD                   | GitHub Actions              | Automated deployments         |

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
│                              AWS VPC                                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│                                                                                  │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                     Application Load Balancer (ALB)                        │  │
│  │                     - SSL Termination                                      │  │
│  │                     - Path-based routing                                   │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                           │
│                                      ▼                                           │
│  ┌───────────────────────────────────────────────────────────────────────────┐  │
│  │                         ECS CLUSTER (Fargate)                              │  │
│  │                                                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     API SERVICE (Node.js/TypeScript)                 │  │  │
│  │  │  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐   │  │  │
│  │  │  │  Instance 1 │ │  Instance 2 │ │  Instance 3 │ │  Instance N │   │  │  │
│  │  │  │  - REST API │ │  - REST API │ │  - REST API │ │  - REST API │   │  │  │
│  │  │  │  - Auth     │ │  - Auth     │ │  - Auth     │ │  - Auth     │   │  │  │
│  │  │  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘   │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  │                                                                            │  │
│  │  ┌──────────────────────┐  ┌──────────────────────┐  ┌─────────────────┐  │  │
│  │  │  DATA AGGREGATION    │  │  PORTFOLIO TRACKER   │  │ EXECUTION       │  │  │
│  │  │  SERVICE (Python)    │  │  SERVICE (Node.js)   │  │ ENGINE (Go)     │  │  │
│  │  │                      │  │                      │  │                 │  │  │
│  │  │  - Protocol scrapers │  │  - Event indexing    │  │ - Fund alloc    │  │  │
│  │  │  - Data normalizer   │  │  - Position tracking │  │ - Rebalancing   │  │  │
│  │  │  - Risk calculator   │  │  - PnL computation   │  │ - TEE secured   │  │  │
│  │  └──────────────────────┘  └──────────────────────┘  └─────────────────┘  │  │
│  │                                                                            │  │
│  │  ┌─────────────────────────────────────────────────────────────────────┐  │  │
│  │  │                     VAULT DEPLOYER BOT (Node.js)                     │  │  │
│  │  │  - Factory contract interaction                                      │  │  │
│  │  │  - Vault configuration                                               │  │  │
│  │  │  - Strategy setup                                                    │  │  │
│  │  └─────────────────────────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────────────────────────┘  │
│                                      │                                           │
│              ┌───────────────────────┼───────────────────────┐                  │
│              ▼                       ▼                       ▼                  │
│  ┌─────────────────────┐ ┌─────────────────────┐ ┌─────────────────────┐       │
│  │   REDIS CLUSTER     │ │   POSTGRESQL        │ │   TIMESCALEDB       │       │
│  │   (ElastiCache)     │ │   (RDS)             │ │   (RDS)             │       │
│  │                     │ │                     │ │                     │       │
│  │   - Cache layer     │ │   - users           │ │   - strategy_metrics│       │
│  │   - Rate limiting   │ │   - neobanks        │ │   - vault_metrics   │       │
│  │   - Session store   │ │   - vaults          │ │   - token_prices    │       │
│  │   - Message queue   │ │   - strategies      │ │   - portfolio_snaps │       │
│  │   - Pub/Sub         │ │   - allocations     │ │   - apy_history     │       │
│  └─────────────────────┘ └─────────────────────┘ └─────────────────────┘       │
│                                                                                  │
└─────────────────────────────────────────────────────────────────────────────────┘
                                       │
                                       ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           EXTERNAL DATA SOURCES                                  │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │  DeFiLlama  │ │  Aavescan   │ │   Morpho    │ │   Pendle    │               │
│  │     API     │ │     API     │ │  GraphQL    │ │    API      │               │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                                  │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐               │
│  │    Lido     │ │   Euler     │ │   Spectra   │ │  CoinGecko  │               │
│  │     API     │ │  Subgraph   │ │  Vision API │ │  (prices)   │               │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘               │
│                                                                                  │
│  ┌─────────────────────────────────────────────────────────────────────────┐   │
│  │                         BLOCKCHAIN RPC NODES                             │   │
│  │   Ethereum (Alchemy/Infura) │ Base │ BSC │ Arbitrum                     │   │
│  └─────────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## Backend Services

### 1. API Service (Node.js/TypeScript)

**Purpose**: Main REST API serving all client requests

**Responsibilities**:

- REST API endpoints (public and authenticated)
- API key authentication and validation
- Rate limiting (per API key)
- Request validation and sanitization
- Response caching
- Unified error handling
- Webhook dispatch

**Tech Stack**:

- Framework: Hono or nest.js
- Validation: Zod
- ORM: Prisma or Drizzle
- API Docs: Swagger

### 2. Data Aggregation Service (Python)

**Purpose**: Fetch, normalize, and store protocol data

**Responsibilities**:

- Protocol-specific scrapers/adapters
- Data normalization (APY/APR standardization)
- Risk score computation
- Time-series data storage
- Fallback handling (DeFiLlama)

**Components**:

```
data-aggregation/
├── scrapers/
│   ├── aave_scraper.py
│   ├── morpho_scraper.py
│   ├── pendle_scraper.py
│   ├── euler_scraper.py
│   ├── spectra_scraper.py
│   └── lido_scraper.py
├── normalizers/
│   ├── apy_normalizer.py
│   ├── token_normalizer.py
│   └── risk_calculator.py
├── scheduler/
│   └── job_scheduler.py
└── storage/
    └── timeseries_writer.py
```

### 3. Portfolio Tracking Service (Node.js/TypeScript)

**Purpose**: Track user positions and compute PnL

**Responsibilities**:

- On-chain event indexing (deposits, withdrawals)
- Real-time position tracking
- Balance aggregation across chains
- PnL computation
- Cost basis tracking
- Portfolio snapshots

### 4. Execution Engine (Go)

**Purpose**: Execute vault operations securely

**Responsibilities**:

- Fund allocation to strategies
- Withdrawal processing
- Rebalancing based on curator allocations
- Transaction signing (TEE secured)
- Nonce management
- Gas optimization

**Security**: Runs in Trusted Execution Environment (TEE)

### 5. Vault Deployer Bot (Node.js/TypeScript)

**Purpose**: Automate vault deployment for neobanks

**Responsibilities**:

- Interact with vault factory contract
- Deploy new vault instances
- Configure vault parameters
- Set up strategy allocations
- Register vault in database
- Generate API keys for neobank

---

## Data Sources

### Protocol Data Sources

| Protocol    | Primary Source   | Endpoint                                             | Update Freq | Rate Limit       |
| ----------- | ---------------- | ---------------------------------------------------- | ----------- | ---------------- |
| **Aave**    | Aavescan API     | `https://api.aavescan.com/v2/reserves/latest`        | 5 min       | API key required |
| **Morpho**  | Morpho GraphQL   | `https://api.morpho.org/graphql`                     | 5 min       | 5k req/5min      |
| **Pendle**  | Pendle V2 API    | `https://api-v2.pendle.finance/core/`                | 5 min       | 100 CU/min       |
| **Euler**   | Goldsky Subgraph | Euler subgraph endpoint                              | 5-10 min    | Generous         |
| **Spectra** | Vision API       | `vision.perspective.fi`                              | 10 min      | TBD              |
| **Lido**    | Lido API         | `https://eth-api.lido.fi/v1/protocol/steth/apr/last` | 30 min      | Generous         |

### Fallback & Aggregated Data

| Source           | Purpose           | Endpoint                            |
| ---------------- | ----------------- | ----------------------------------- |
| DeFiLlama        | TVL, APY fallback | `https://api.llama.fi/`             |
| DeFiLlama Yields | Yield data        | `https://yields.llama.fi/`          |
| CoinGecko        | Token prices      | `https://api.coingecko.com/api/v3/` |

### RPC Providers

| Chain    | Primary   | Fallback  |
| -------- | --------- | --------- |
| Ethereum | Alchemy   | Infura    |
| Base     | Alchemy   | QuickNode |
| BSC      | QuickNode | NodeReal  |
| Arbitrum | Alchemy   | Infura    |

---

## Database Design

### PostgreSQL Schema

```sql
-- =====================
-- CORE ENTITIES
-- =====================

-- Neobank accounts (B2B customers)
CREATE TABLE neobanks (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(255) NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    api_key VARCHAR(64) UNIQUE NOT NULL,
    api_key_hash VARCHAR(256) NOT NULL,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);

-- End users (neobank customers)
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    neobank_id UUID REFERENCES neobanks(id),
    external_id VARCHAR(255), -- neobank's user identifier
    wallet_address VARCHAR(42),
    email VARCHAR(255),
    name VARCHAR(255),
    is_verified BOOLEAN DEFAULT false,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(neobank_id, external_id)
);

-- Supported blockchains
CREATE TABLE chains (
    id INTEGER PRIMARY KEY, -- chain ID
    name VARCHAR(50) NOT NULL,
    rpc_url VARCHAR(255) NOT NULL,
    explorer_url VARCHAR(255),
    is_active BOOLEAN DEFAULT true,
    block_time_ms INTEGER DEFAULT 12000
);

-- DeFi protocols
CREATE TABLE protocols (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    name VARCHAR(100) NOT NULL,
    slug VARCHAR(50) UNIQUE NOT NULL,
    logo_url VARCHAR(255),
    website_url VARCHAR(255),
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Tokens
CREATE TABLE tokens (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    chain_id INTEGER REFERENCES chains(id),
    address VARCHAR(42) NOT NULL,
    symbol VARCHAR(20) NOT NULL,
    name VARCHAR(100),
    decimals INTEGER NOT NULL,
    logo_url VARCHAR(255),
    coingecko_id VARCHAR(100),
    UNIQUE(chain_id, address)
);

-- =====================
-- STRATEGIES
-- =====================

-- Yield strategies (pools, vaults from protocols)
CREATE TABLE strategies (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    identifier VARCHAR(100) UNIQUE NOT NULL, -- external identifier
    protocol_id UUID REFERENCES protocols(id),
    chain_id INTEGER REFERENCES chains(id),
    name VARCHAR(255) NOT NULL,
    description TEXT,
    strategy_type VARCHAR(50) NOT NULL, -- 'lending', 'staking', 'lp', 'vault', etc.
    contract_address VARCHAR(42) NOT NULL,
    deposit_token_id UUID REFERENCES tokens(id),

    -- Current metrics (updated by scraper)
    current_apy DECIMAL(10, 4),
    reward_apy DECIMAL(10, 4),
    total_apy DECIMAL(10, 4),
    tvl_usd DECIMAL(20, 2),
    utilization_rate DECIMAL(5, 4),

    -- Risk metrics
    risk_score INTEGER CHECK (risk_score BETWEEN 1 AND 10),
    risk_profile VARCHAR(20), -- 'low', 'medium', 'high'

    -- Metadata
    is_active BOOLEAN DEFAULT true,
    last_updated TIMESTAMPTZ,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =====================
-- VAULTS
-- =====================

-- Neobank vaults (deployed via factory)
CREATE TABLE vaults (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    neobank_id UUID REFERENCES neobanks(id),
    chain_id INTEGER REFERENCES chains(id),

    -- Contract info
    vault_address VARCHAR(42) NOT NULL,
    bot_address VARCHAR(42) NOT NULL,

    -- Metadata
    name VARCHAR(255) NOT NULL,
    description TEXT,
    vault_type VARCHAR(50),

    -- Current metrics
    current_apy DECIMAL(10, 4),
    tvl_usd DECIMAL(20, 2),
    share_price DECIMAL(30, 18),
    total_shares DECIMAL(30, 18),

    -- Risk
    risk_profile VARCHAR(20),

    -- Status
    status VARCHAR(20) DEFAULT 'active', -- 'deploying', 'active', 'paused', 'deprecated'
    is_testnet BOOLEAN DEFAULT false,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(chain_id, vault_address)
);

-- Vault strategy allocations
CREATE TABLE vault_allocations (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    vault_id UUID REFERENCES vaults(id),
    strategy_id UUID REFERENCES strategies(id),
    allocation_percentage DECIMAL(5, 2) NOT NULL CHECK (allocation_percentage >= 0 AND allocation_percentage <= 100),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),
    UNIQUE(vault_id, strategy_id)
);

-- =====================
-- USER POSITIONS
-- =====================

-- User deposits/positions
CREATE TABLE user_positions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    vault_id UUID REFERENCES vaults(id),

    -- Position data
    shares DECIMAL(30, 18) NOT NULL DEFAULT 0,
    cost_basis_usd DECIMAL(20, 2) DEFAULT 0,

    -- Tracking
    total_deposited_usd DECIMAL(20, 2) DEFAULT 0,
    total_withdrawn_usd DECIMAL(20, 2) DEFAULT 0,

    first_deposit_at TIMESTAMPTZ,
    last_activity_at TIMESTAMPTZ,

    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW(),

    UNIQUE(user_id, vault_id)
);

-- Transaction history
CREATE TABLE transactions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id UUID REFERENCES users(id),
    vault_id UUID REFERENCES vaults(id),
    chain_id INTEGER REFERENCES chains(id),

    -- Transaction details
    tx_hash VARCHAR(66) NOT NULL,
    tx_type VARCHAR(20) NOT NULL, -- 'deposit', 'withdraw', 'claim'
    status VARCHAR(20) DEFAULT 'pending', -- 'pending', 'confirmed', 'failed'

    -- Amounts
    amount DECIMAL(30, 18),
    amount_usd DECIMAL(20, 2),
    shares DECIMAL(30, 18),
    share_price DECIMAL(30, 18),

    -- Timestamps
    submitted_at TIMESTAMPTZ DEFAULT NOW(),
    confirmed_at TIMESTAMPTZ,
    block_number BIGINT,

    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- =====================
-- INDEXES
-- =====================

CREATE INDEX idx_strategies_protocol ON strategies(protocol_id);
CREATE INDEX idx_strategies_chain ON strategies(chain_id);
CREATE INDEX idx_strategies_type ON strategies(strategy_type);
CREATE INDEX idx_strategies_active ON strategies(is_active) WHERE is_active = true;

CREATE INDEX idx_vaults_neobank ON vaults(neobank_id);
CREATE INDEX idx_vaults_chain ON vaults(chain_id);
CREATE INDEX idx_vaults_status ON vaults(status);

CREATE INDEX idx_user_positions_user ON user_positions(user_id);
CREATE INDEX idx_user_positions_vault ON user_positions(vault_id);

CREATE INDEX idx_transactions_user ON transactions(user_id);
CREATE INDEX idx_transactions_vault ON transactions(vault_id);
CREATE INDEX idx_transactions_hash ON transactions(tx_hash);
CREATE INDEX idx_transactions_status ON transactions(status);
```

### TimescaleDB Schema

```sql
-- Enable TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- =====================
-- TIME-SERIES TABLES
-- =====================

-- Strategy metrics over time
CREATE TABLE strategy_metrics (
    time TIMESTAMPTZ NOT NULL,
    strategy_id UUID NOT NULL,
    apy DECIMAL(10, 4),
    reward_apy DECIMAL(10, 4),
    total_apy DECIMAL(10, 4),
    tvl_usd DECIMAL(20, 2),
    utilization_rate DECIMAL(5, 4),
    borrow_apy DECIMAL(10, 4),
    supply_apy DECIMAL(10, 4)
);

SELECT create_hypertable('strategy_metrics', 'time');
CREATE INDEX idx_strategy_metrics_strategy ON strategy_metrics(strategy_id, time DESC);

-- Vault metrics over time
CREATE TABLE vault_metrics (
    time TIMESTAMPTZ NOT NULL,
    vault_id UUID NOT NULL,
    apy DECIMAL(10, 4),
    tvl_usd DECIMAL(20, 2),
    share_price DECIMAL(30, 18),
    total_shares DECIMAL(30, 18)
);

SELECT create_hypertable('vault_metrics', 'time');
CREATE INDEX idx_vault_metrics_vault ON vault_metrics(vault_id, time DESC);

-- Token prices
CREATE TABLE token_prices (
    time TIMESTAMPTZ NOT NULL,
    token_id UUID NOT NULL,
    price_usd DECIMAL(20, 8) NOT NULL
);

SELECT create_hypertable('token_prices', 'time');
CREATE INDEX idx_token_prices_token ON token_prices(token_id, time DESC);

-- User portfolio snapshots (daily)
CREATE TABLE portfolio_snapshots (
    time TIMESTAMPTZ NOT NULL,
    user_id UUID NOT NULL,
    vault_id UUID NOT NULL,
    shares DECIMAL(30, 18),
    value_usd DECIMAL(20, 2),
    cost_basis_usd DECIMAL(20, 2),
    pnl_usd DECIMAL(20, 2),
    pnl_percentage DECIMAL(10, 4)
);

SELECT create_hypertable('portfolio_snapshots', 'time');
CREATE INDEX idx_portfolio_snapshots_user ON portfolio_snapshots(user_id, time DESC);

-- =====================
-- RETENTION POLICIES
-- =====================

-- Keep detailed metrics for 90 days, then downsample
SELECT add_retention_policy('strategy_metrics', INTERVAL '90 days');
SELECT add_retention_policy('token_prices', INTERVAL '90 days');

-- Keep vault metrics for 1 year
SELECT add_retention_policy('vault_metrics', INTERVAL '365 days');

-- Keep portfolio snapshots for 2 years
SELECT add_retention_policy('portfolio_snapshots', INTERVAL '730 days');

-- =====================
-- CONTINUOUS AGGREGATES (for faster queries)
-- =====================

-- Hourly strategy metrics
CREATE MATERIALIZED VIEW strategy_metrics_hourly
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 hour', time) AS bucket,
    strategy_id,
    AVG(apy) as avg_apy,
    AVG(total_apy) as avg_total_apy,
    AVG(tvl_usd) as avg_tvl,
    MAX(tvl_usd) as max_tvl,
    MIN(tvl_usd) as min_tvl
FROM strategy_metrics
GROUP BY bucket, strategy_id;

-- Daily strategy metrics
CREATE MATERIALIZED VIEW strategy_metrics_daily
WITH (timescaledb.continuous) AS
SELECT
    time_bucket('1 day', time) AS bucket,
    strategy_id,
    AVG(apy) as avg_apy,
    AVG(total_apy) as avg_total_apy,
    AVG(tvl_usd) as avg_tvl,
    MAX(tvl_usd) as max_tvl,
    MIN(tvl_usd) as min_tvl,
    FIRST(apy, time) as open_apy,
    LAST(apy, time) as close_apy
FROM strategy_metrics
GROUP BY bucket, strategy_id;
```

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

### AWS Infrastructure

```
┌─────────────────────────────────────────────────────────────────┐
│                        AWS REGION                                │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │                        VPC                               │    │
│  │  ┌─────────────────┐        ┌─────────────────┐         │    │
│  │  │  Public Subnet  │        │  Public Subnet  │         │    │
│  │  │    (AZ-1)       │        │    (AZ-2)       │         │    │
│  │  │  ┌───────────┐  │        │  ┌───────────┐  │         │    │
│  │  │  │    ALB    │  │        │  │    ALB    │  │         │    │
│  │  │  └───────────┘  │        │  └───────────┘  │         │    │
│  │  └─────────────────┘        └─────────────────┘         │    │
│  │                                                          │    │
│  │  ┌─────────────────┐        ┌─────────────────┐         │    │
│  │  │ Private Subnet  │        │ Private Subnet  │         │    │
│  │  │    (AZ-1)       │        │    (AZ-2)       │         │    │
│  │  │                 │        │                 │         │    │
│  │  │  ┌───────────┐  │        │  ┌───────────┐  │         │    │
│  │  │  │ECS Fargate│  │        │  │ECS Fargate│  │         │    │
│  │  │  │ Services  │  │        │  │ Services  │  │         │    │
│  │  │  └───────────┘  │        │  └───────────┘  │         │    │
│  │  └─────────────────┘        └─────────────────┘         │    │
│  │                                                          │    │
│  │  ┌─────────────────┐        ┌─────────────────┐         │    │
│  │  │  Data Subnet    │        │  Data Subnet    │         │    │
│  │  │    (AZ-1)       │        │    (AZ-2)       │         │    │
│  │  │                 │        │                 │         │    │
│  │  │  ┌───────────┐  │        │  ┌───────────┐  │         │    │
│  │  │  │    RDS    │  │        │  │   Redis   │  │         │    │
│  │  │  │ (Primary) │◄─┼────────┼─►│ElastiCache│  │         │    │
│  │  │  └───────────┘  │        │  └───────────┘  │         │    │
│  │  └─────────────────┘        └─────────────────┘         │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Route 53   │  │     ECR      │  │  CloudWatch  │           │
│  │    (DNS)     │  │  (Images)    │  │ (Monitoring) │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│                                                                  │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │   Secrets    │  │     S3       │  │    WAF       │           │
│  │   Manager    │  │  (Backups)   │  │ (Security)   │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
└─────────────────────────────────────────────────────────────────┘
```

### Environment Configuration

| Environment | Purpose           | Database         | Cache                 |
| ----------- | ----------------- | ---------------- | --------------------- |
| Development | Local development | Local PostgreSQL | Local Redis           |
| Staging     | Testing, QA       | RDS (small)      | ElastiCache (small)   |
| Production  | Live traffic      | RDS (Multi-AZ)   | ElastiCache (Cluster) |

### CI/CD Pipeline

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main, staging]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build Docker images
      - name: Push to ECR
      - name: Update ECS service
```

---

## Next Steps

1. **Phase 1**: Core Backend Setup

   - [ ] Set up AWS infrastructure (VPC, RDS, ElastiCache, ECS)
   - [ ] Implement API service with core endpoints
   - [ ] Build data aggregation service with protocol scrapers

2. **Phase 2**: Vault System

   - [ ] Develop vault factory smart contracts
   - [ ] Implement vault deployer bot
   - [ ] Build execution engine

3. **Phase 3**: SDK Development

   - [ ] Design SDK architecture
   - [ ] Implement core modules
   - [ ] Write documentation and examples

4. **Phase 4**: Testing & Launch
   - [ ] Integration testing
   - [ ] Security audit
   - [ ] Testnet deployment
   - [ ] Mainnet launch

---

## References

- [DeFiLlama API](https://defillama.com/docs/api)
- [vaults.fyi Documentation](https://docs.vaults.fyi)
- [Morpho API](https://docs.morpho.org/tools/offchain/api/get-started/)
- [Pendle API](https://docs.pendle.finance/pendle-v2/Developers/Backend/ApiOverview)
- [Euler Subgraphs](https://docs.euler.finance/developers/data-querying/subgraphs/)
- [Lido API](https://docs.lido.fi/integrations/api/)
- [Spectra Developer Docs](https://dev.spectra.finance/)
- [Veda Boring Vault](https://docs.vaults.fyi)
