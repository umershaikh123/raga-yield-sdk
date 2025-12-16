# Raga Finance - Project Structure

## Monorepo Architecture

This document defines the complete project structure for the Raga Finance yield infrastructure platform, following senior engineering best practices.

---

## Table of Contents

1. [Overview](#overview)
2. [Monorepo Structure](#monorepo-structure)
3. [Apps Breakdown](#apps-breakdown)
4. [Packages Breakdown](#packages-breakdown)
5. [NestJS Backend Architecture](#nestjs-backend-architecture)
6. [SDK Architecture](#sdk-architecture)
7. [Configuration Management](#configuration-management)
8. [Dependency Graph](#dependency-graph)

---

## Overview

### Tech Stack Summary

| Layer | Technology | Purpose |
|-------|------------|---------|
| **Monorepo** | Turborepo + pnpm | Build orchestration, caching |
| **API Backend** | NestJS + TypeScript | REST API, business logic |
| **SDK** | TypeScript | Client library |
| **ORM** | Prisma | Database access, migrations |
| **Validation** | Zod + class-validator | Schema validation |
| **Database** | PostgreSQL + TimescaleDB | Data persistence |
| **Cache** | Redis | Caching, rate limiting, queues |
| **Testing** | Jest + Supertest | Unit, integration, e2e tests |
| **Documentation** | OpenAPI/Swagger | API documentation |

### Design Principles Applied

- **Domain-Driven Design (DDD)** - Business logic organized by domain
- **Clean Architecture** - Dependency inversion, separation of concerns
- **SOLID Principles** - Single responsibility, open/closed, etc.
- **Repository Pattern** - Data access abstraction
- **CQRS-lite** - Separate read/write models where beneficial
- **Hexagonal Architecture** - Ports and adapters for external systems

---

## Monorepo Structure

```
raga-finance/
├── .github/
│   ├── workflows/
│   │   ├── ci.yml                    # CI pipeline
│   │   ├── cd-staging.yml            # Deploy to staging
│   │   ├── cd-production.yml         # Deploy to production
│   │   └── release.yml               # SDK release pipeline
│   ├── CODEOWNERS
│   └── pull_request_template.md
│
├── apps/
│   ├── api/                          # NestJS API backend
│   ├── data-aggregator/              # Python data scraping service
│   ├── execution-engine/             # Go execution engine
│   └── vault-deployer/               # Vault deployment bot
│
├── packages/
│   ├── sdk/                          # TypeScript SDK (@raga-finance/sdk)
│   ├── types/                        # Shared TypeScript types (@raga-finance/types)
│   ├── contracts/                    # Smart contract ABIs & types
│   ├── config/                       # Shared configuration schemas
│   ├── eslint-config/                # ESLint configurations
│   └── tsconfig/                     # TypeScript configurations
│
├── infrastructure/
│   ├── terraform/                    # AWS infrastructure as code
│   ├── docker/                       # Docker configurations
│   └── k8s/ (optional)               # Kubernetes manifests
│
├── docs/
│   ├── architecture/                 # Architecture decision records
│   ├── api/                          # API documentation
│   └── sdk/                          # SDK documentation
│
├── scripts/
│   ├── setup.sh                      # Development setup
│   ├── seed-db.ts                    # Database seeding
│   └── generate-types.ts             # Type generation
│
├── .env.example                      # Environment variables template
├── .gitignore
├── .npmrc
├── docker-compose.yml                # Local development services
├── docker-compose.prod.yml           # Production services
├── package.json                      # Root package.json
├── pnpm-workspace.yaml               # pnpm workspace config
├── turbo.json                        # Turborepo config
├── tsconfig.base.json                # Base TypeScript config
├── ARCHITECTURE.md
├── API_SPECIFICATION.md
├── PROJECT_STRUCTURE.md
├── DESIGN_PRINCIPLES.md
└── README.md
```

---

## Apps Breakdown

### 1. API Backend (`apps/api`)

NestJS application serving the REST API.

```
apps/api/
├── src/
│   ├── main.ts                       # Application entry point
│   ├── app.module.ts                 # Root module
│   │
│   ├── common/                       # Shared utilities
│   │   ├── decorators/
│   │   │   ├── api-key.decorator.ts
│   │   │   ├── user.decorator.ts
│   │   │   └── public.decorator.ts
│   │   ├── filters/
│   │   │   ├── http-exception.filter.ts
│   │   │   └── all-exceptions.filter.ts
│   │   ├── guards/
│   │   │   ├── api-key.guard.ts
│   │   │   ├── rate-limit.guard.ts
│   │   │   └── roles.guard.ts
│   │   ├── interceptors/
│   │   │   ├── logging.interceptor.ts
│   │   │   ├── transform.interceptor.ts
│   │   │   └── cache.interceptor.ts
│   │   ├── pipes/
│   │   │   └── validation.pipe.ts
│   │   ├── middleware/
│   │   │   ├── request-id.middleware.ts
│   │   │   └── logger.middleware.ts
│   │   └── utils/
│   │       ├── pagination.util.ts
│   │       └── crypto.util.ts
│   │
│   ├── config/                       # Configuration module
│   │   ├── config.module.ts
│   │   ├── configuration.ts          # Config loader
│   │   ├── validation.ts             # Env validation schema
│   │   └── configs/
│   │       ├── app.config.ts
│   │       ├── database.config.ts
│   │       ├── redis.config.ts
│   │       ├── auth.config.ts
│   │       └── blockchain.config.ts
│   │
│   ├── database/                     # Database module
│   │   ├── database.module.ts
│   │   ├── prisma/
│   │   │   ├── prisma.module.ts
│   │   │   ├── prisma.service.ts
│   │   │   └── prisma.health.ts
│   │   └── timescale/
│   │       ├── timescale.module.ts
│   │       └── timescale.service.ts
│   │
│   ├── cache/                        # Redis/Cache module
│   │   ├── cache.module.ts
│   │   ├── cache.service.ts
│   │   └── cache.keys.ts
│   │
│   ├── queue/                        # Queue module (Redis Streams/BullMQ)
│   │   ├── queue.module.ts
│   │   ├── producers/
│   │   │   ├── vault-deploy.producer.ts
│   │   │   └── notification.producer.ts
│   │   └── consumers/
│   │       ├── vault-deploy.consumer.ts
│   │       └── notification.consumer.ts
│   │
│   ├── health/                       # Health checks
│   │   ├── health.module.ts
│   │   └── health.controller.ts
│   │
│   │── ─────────────────────────────────────────────────────
│   │   DOMAIN MODULES (DDD-style)
│   │── ─────────────────────────────────────────────────────
│   │
│   ├── modules/
│   │   │
│   │   ├── auth/                     # Authentication domain
│   │   │   ├── auth.module.ts
│   │   │   ├── auth.controller.ts
│   │   │   ├── auth.service.ts
│   │   │   ├── strategies/
│   │   │   │   └── api-key.strategy.ts
│   │   │   └── dto/
│   │   │       └── auth.dto.ts
│   │   │
│   │   ├── neobanks/                 # Neobank domain
│   │   │   ├── neobanks.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── neobanks.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── neobanks.service.ts
│   │   │   │   └── api-key.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── neobanks.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-neobank.dto.ts
│   │   │   │   └── neobank-response.dto.ts
│   │   │   └── entities/
│   │   │       └── neobank.entity.ts
│   │   │
│   │   ├── strategies/               # Yield Strategies domain
│   │   │   ├── strategies.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── strategies.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── strategies.service.ts
│   │   │   │   ├── strategy-metrics.service.ts
│   │   │   │   └── strategy-sync.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── strategies.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── get-strategies.dto.ts
│   │   │   │   ├── strategy-response.dto.ts
│   │   │   │   └── strategy-details.dto.ts
│   │   │   ├── entities/
│   │   │   │   └── strategy.entity.ts
│   │   │   └── interfaces/
│   │   │       └── strategy.interface.ts
│   │   │
│   │   ├── protocols/                # DeFi Protocols domain
│   │   │   ├── protocols.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── protocols.controller.ts
│   │   │   ├── services/
│   │   │   │   └── protocols.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── protocols.repository.ts
│   │   │   └── dto/
│   │   │       └── protocol-response.dto.ts
│   │   │
│   │   ├── chains/                   # Blockchain Chains domain
│   │   │   ├── chains.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── chains.controller.ts
│   │   │   ├── services/
│   │   │   │   └── chains.service.ts
│   │   │   └── dto/
│   │   │       └── chain-response.dto.ts
│   │   │
│   │   ├── vaults/                   # Vaults domain
│   │   │   ├── vaults.module.ts
│   │   │   ├── controllers/
│   │   │   │   ├── vaults.controller.ts
│   │   │   │   └── vault-admin.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── vaults.service.ts
│   │   │   │   ├── vault-deployment.service.ts
│   │   │   │   ├── vault-metrics.service.ts
│   │   │   │   └── vault-allocation.service.ts
│   │   │   ├── repositories/
│   │   │   │   ├── vaults.repository.ts
│   │   │   │   └── vault-allocations.repository.ts
│   │   │   ├── dto/
│   │   │   │   ├── create-vault.dto.ts
│   │   │   │   ├── update-vault.dto.ts
│   │   │   │   └── vault-response.dto.ts
│   │   │   ├── entities/
│   │   │   │   ├── vault.entity.ts
│   │   │   │   └── vault-allocation.entity.ts
│   │   │   └── events/
│   │   │       ├── vault-created.event.ts
│   │   │       └── vault-rebalanced.event.ts
│   │   │
│   │   ├── portfolio/                # Portfolio domain
│   │   │   ├── portfolio.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── portfolio.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── portfolio.service.ts
│   │   │   │   ├── positions.service.ts
│   │   │   │   └── pnl.service.ts
│   │   │   ├── repositories/
│   │   │   │   ├── positions.repository.ts
│   │   │   │   └── portfolio-snapshots.repository.ts
│   │   │   └── dto/
│   │   │       ├── portfolio-response.dto.ts
│   │   │       ├── position-response.dto.ts
│   │   │       └── pnl-response.dto.ts
│   │   │
│   │   ├── transactions/             # Transactions domain
│   │   │   ├── transactions.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── transactions.controller.ts
│   │   │   ├── services/
│   │   │   │   ├── transactions.service.ts
│   │   │   │   ├── transaction-builder.service.ts
│   │   │   │   └── transaction-simulator.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── transactions.repository.ts
│   │   │   └── dto/
│   │   │       ├── prepare-deposit.dto.ts
│   │   │       ├── prepare-withdraw.dto.ts
│   │   │       └── transaction-response.dto.ts
│   │   │
│   │   ├── users/                    # Users domain
│   │   │   ├── users.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── users.controller.ts
│   │   │   ├── services/
│   │   │   │   └── users.service.ts
│   │   │   ├── repositories/
│   │   │   │   └── users.repository.ts
│   │   │   └── dto/
│   │   │       ├── create-user.dto.ts
│   │   │       └── user-response.dto.ts
│   │   │
│   │   ├── benchmarks/               # Market Benchmarks domain
│   │   │   ├── benchmarks.module.ts
│   │   │   ├── controllers/
│   │   │   │   └── benchmarks.controller.ts
│   │   │   └── services/
│   │   │       └── benchmarks.service.ts
│   │   │
│   │   └── webhooks/                 # Webhooks domain
│   │       ├── webhooks.module.ts
│   │       ├── services/
│   │       │   ├── webhooks.service.ts
│   │       │   └── webhook-dispatcher.service.ts
│   │       └── dto/
│   │           └── webhook-config.dto.ts
│   │
│   │── ─────────────────────────────────────────────────────
│   │   INFRASTRUCTURE / EXTERNAL ADAPTERS
│   │── ─────────────────────────────────────────────────────
│   │
│   ├── infrastructure/
│   │   │
│   │   ├── blockchain/               # Blockchain interactions
│   │   │   ├── blockchain.module.ts
│   │   │   ├── providers/
│   │   │   │   ├── ethereum.provider.ts
│   │   │   │   ├── base.provider.ts
│   │   │   │   ├── bsc.provider.ts
│   │   │   │   └── arbitrum.provider.ts
│   │   │   ├── services/
│   │   │   │   ├── multicall.service.ts
│   │   │   │   ├── contract-reader.service.ts
│   │   │   │   └── gas-estimator.service.ts
│   │   │   └── abis/
│   │   │       ├── erc20.abi.ts
│   │   │       ├── vault.abi.ts
│   │   │       └── vault-factory.abi.ts
│   │   │
│   │   ├── external-apis/            # External API clients
│   │   │   ├── external-apis.module.ts
│   │   │   ├── defillama/
│   │   │   │   ├── defillama.service.ts
│   │   │   │   └── defillama.types.ts
│   │   │   ├── aavescan/
│   │   │   │   ├── aavescan.service.ts
│   │   │   │   └── aavescan.types.ts
│   │   │   ├── morpho/
│   │   │   │   ├── morpho.service.ts
│   │   │   │   └── morpho.types.ts
│   │   │   ├── pendle/
│   │   │   │   ├── pendle.service.ts
│   │   │   │   └── pendle.types.ts
│   │   │   ├── lido/
│   │   │   │   ├── lido.service.ts
│   │   │   │   └── lido.types.ts
│   │   │   └── coingecko/
│   │   │       ├── coingecko.service.ts
│   │   │       └── coingecko.types.ts
│   │   │
│   │   └── notifications/            # Notification services
│   │       ├── notifications.module.ts
│   │       ├── email.service.ts
│   │       └── slack.service.ts
│   │
│   └── shared/                       # Shared constants, enums
│       ├── constants/
│       │   ├── chains.constant.ts
│       │   ├── protocols.constant.ts
│       │   └── errors.constant.ts
│       └── enums/
│           ├── chain.enum.ts
│           ├── protocol.enum.ts
│           ├── strategy-type.enum.ts
│           └── transaction-status.enum.ts
│
├── prisma/
│   ├── schema.prisma                 # Prisma schema
│   ├── migrations/                   # Database migrations
│   └── seed.ts                       # Seed data
│
├── test/
│   ├── unit/                         # Unit tests
│   │   └── modules/
│   │       ├── strategies/
│   │       ├── vaults/
│   │       └── portfolio/
│   ├── integration/                  # Integration tests
│   │   └── modules/
│   │       ├── strategies.integration.spec.ts
│   │       └── vaults.integration.spec.ts
│   ├── e2e/                          # End-to-end tests
│   │   ├── strategies.e2e-spec.ts
│   │   ├── vaults.e2e-spec.ts
│   │   └── portfolio.e2e-spec.ts
│   ├── fixtures/                     # Test fixtures
│   │   ├── strategies.fixture.ts
│   │   ├── vaults.fixture.ts
│   │   └── users.fixture.ts
│   ├── mocks/                        # Mock services
│   │   ├── prisma.mock.ts
│   │   ├── redis.mock.ts
│   │   └── blockchain.mock.ts
│   └── setup.ts                      # Test setup
│
├── .env.example
├── .env.test
├── Dockerfile
├── Dockerfile.dev
├── nest-cli.json
├── package.json
├── tsconfig.json
├── tsconfig.build.json
└── README.md
```

---

### 2. Data Aggregator (`apps/data-aggregator`)

Python service for scraping and normalizing protocol data.

```
apps/data-aggregator/
├── src/
│   ├── main.py                       # Entry point
│   ├── config/
│   │   ├── __init__.py
│   │   ├── settings.py               # Configuration
│   │   └── logging.py                # Logging setup
│   │
│   ├── scrapers/                     # Protocol scrapers
│   │   ├── __init__.py
│   │   ├── base_scraper.py           # Abstract base class
│   │   ├── aave_scraper.py
│   │   ├── morpho_scraper.py
│   │   ├── pendle_scraper.py
│   │   ├── euler_scraper.py
│   │   ├── spectra_scraper.py
│   │   ├── lido_scraper.py
│   │   └── defillama_scraper.py      # Fallback
│   │
│   ├── normalizers/                  # Data normalization
│   │   ├── __init__.py
│   │   ├── apy_normalizer.py
│   │   ├── token_normalizer.py
│   │   └── risk_calculator.py
│   │
│   ├── storage/                      # Database writers
│   │   ├── __init__.py
│   │   ├── postgres_writer.py
│   │   └── timescale_writer.py
│   │
│   ├── scheduler/                    # Job scheduling
│   │   ├── __init__.py
│   │   └── scheduler.py
│   │
│   └── utils/
│       ├── __init__.py
│       ├── http_client.py
│       └── retry.py
│
├── tests/
│   ├── unit/
│   └── integration/
│
├── Dockerfile
├── requirements.txt
├── pyproject.toml
└── README.md
```

---

### 3. Execution Engine (`apps/execution-engine`)

Go service for secure fund allocation.

```
apps/execution-engine/
├── cmd/
│   └── engine/
│       └── main.go                   # Entry point
│
├── internal/
│   ├── config/
│   │   └── config.go
│   │
│   ├── executor/
│   │   ├── executor.go               # Main executor
│   │   ├── deposit.go
│   │   ├── withdraw.go
│   │   └── rebalance.go
│   │
│   ├── blockchain/
│   │   ├── client.go
│   │   ├── multicall.go
│   │   └── gas.go
│   │
│   ├── vault/
│   │   ├── vault.go
│   │   └── strategy.go
│   │
│   ├── queue/
│   │   ├── consumer.go
│   │   └── producer.go
│   │
│   └── tee/                          # TEE-specific code
│       ├── enclave.go
│       └── attestation.go
│
├── pkg/
│   ├── contracts/                    # Contract bindings
│   └── types/
│
├── Dockerfile
├── go.mod
├── go.sum
└── README.md
```

---

### 4. Vault Deployer (`apps/vault-deployer`)

Node.js bot for automated vault deployment.

```
apps/vault-deployer/
├── src/
│   ├── main.ts
│   ├── config/
│   │   └── config.ts
│   │
│   ├── deployer/
│   │   ├── deployer.service.ts
│   │   ├── factory.service.ts
│   │   └── config-builder.ts
│   │
│   ├── queue/
│   │   └── consumer.ts
│   │
│   └── blockchain/
│       ├── signer.ts
│       └── contracts.ts
│
├── Dockerfile
├── package.json
└── README.md
```

---

## Packages Breakdown

### 1. SDK Package (`packages/sdk`)

TypeScript SDK for API consumers.

```
packages/sdk/
├── src/
│   ├── index.ts                      # Main exports
│   │
│   ├── client/
│   │   ├── RagaClient.ts             # Main client class
│   │   ├── HttpClient.ts             # HTTP wrapper
│   │   ├── types.ts                  # Client types
│   │   └── errors.ts                 # SDK errors
│   │
│   ├── modules/
│   │   ├── strategies/
│   │   │   ├── index.ts
│   │   │   ├── strategies.module.ts
│   │   │   └── strategies.types.ts
│   │   │
│   │   ├── protocols/
│   │   │   ├── index.ts
│   │   │   ├── protocols.module.ts
│   │   │   └── protocols.types.ts
│   │   │
│   │   ├── chains/
│   │   │   ├── index.ts
│   │   │   ├── chains.module.ts
│   │   │   └── chains.types.ts
│   │   │
│   │   ├── vaults/
│   │   │   ├── index.ts
│   │   │   ├── vaults.module.ts
│   │   │   └── vaults.types.ts
│   │   │
│   │   ├── portfolio/
│   │   │   ├── index.ts
│   │   │   ├── portfolio.module.ts
│   │   │   └── portfolio.types.ts
│   │   │
│   │   ├── transactions/
│   │   │   ├── index.ts
│   │   │   ├── transactions.module.ts
│   │   │   └── transactions.types.ts
│   │   │
│   │   └── benchmarks/
│   │       ├── index.ts
│   │       ├── benchmarks.module.ts
│   │       └── benchmarks.types.ts
│   │
│   ├── utils/
│   │   ├── formatters.ts             # APY, currency formatters
│   │   ├── validators.ts             # Input validation
│   │   ├── retry.ts                  # Retry logic
│   │   └── cache.ts                  # Client-side caching
│   │
│   └── types/
│       ├── index.ts
│       ├── common.types.ts
│       └── api-response.types.ts
│
├── tests/
│   ├── unit/
│   │   ├── client.test.ts
│   │   └── modules/
│   └── integration/
│       └── api.integration.test.ts
│
├── examples/
│   ├── basic-usage.ts
│   ├── fetch-strategies.ts
│   ├── create-vault.ts
│   └── portfolio-tracking.ts
│
├── package.json
├── tsconfig.json
├── tsup.config.ts                    # Build configuration
├── vitest.config.ts                  # Test configuration
└── README.md
```

---

### 2. Types Package (`packages/types`)

Shared TypeScript types across all apps.

```
packages/types/
├── src/
│   ├── index.ts
│   │
│   ├── entities/
│   │   ├── neobank.types.ts
│   │   ├── user.types.ts
│   │   ├── strategy.types.ts
│   │   ├── protocol.types.ts
│   │   ├── chain.types.ts
│   │   ├── vault.types.ts
│   │   ├── position.types.ts
│   │   └── transaction.types.ts
│   │
│   ├── api/
│   │   ├── request.types.ts
│   │   ├── response.types.ts
│   │   ├── pagination.types.ts
│   │   └── error.types.ts
│   │
│   ├── blockchain/
│   │   ├── token.types.ts
│   │   ├── contract.types.ts
│   │   └── transaction.types.ts
│   │
│   └── enums/
│       ├── chain.enum.ts
│       ├── protocol.enum.ts
│       ├── strategy-type.enum.ts
│       ├── risk-profile.enum.ts
│       └── transaction-status.enum.ts
│
├── package.json
├── tsconfig.json
└── README.md
```

---

### 3. Contracts Package (`packages/contracts`)

Smart contract ABIs and TypeScript bindings.

```
packages/contracts/
├── src/
│   ├── index.ts
│   │
│   ├── abis/
│   │   ├── ERC20.json
│   │   ├── VaultFactory.json
│   │   ├── Vault.json
│   │   ├── AavePool.json
│   │   └── MorphoBlue.json
│   │
│   ├── addresses/
│   │   ├── mainnet.ts
│   │   ├── base.ts
│   │   ├── bsc.ts
│   │   └── arbitrum.ts
│   │
│   └── types/                        # Generated by typechain
│       ├── ERC20.ts
│       ├── VaultFactory.ts
│       └── Vault.ts
│
├── package.json
├── tsconfig.json
└── README.md
```

---

### 4. Config Package (`packages/config`)

Shared configuration schemas and utilities.

```
packages/config/
├── src/
│   ├── index.ts
│   │
│   ├── schemas/
│   │   ├── env.schema.ts             # Zod schemas for env vars
│   │   ├── database.schema.ts
│   │   └── blockchain.schema.ts
│   │
│   └── utils/
│       ├── env-loader.ts
│       └── config-validator.ts
│
├── package.json
├── tsconfig.json
└── README.md
```

---

### 5. ESLint Config (`packages/eslint-config`)

Shared ESLint configurations.

```
packages/eslint-config/
├── base.js                           # Base config
├── nestjs.js                         # NestJS-specific
├── react.js                          # React (if needed)
└── package.json
```

---

### 6. TypeScript Config (`packages/tsconfig`)

Shared TypeScript configurations.

```
packages/tsconfig/
├── base.json                         # Base config
├── nestjs.json                       # NestJS apps
├── node.json                         # Node.js packages
├── sdk.json                          # SDK build
└── package.json
```

---

## NestJS Backend Architecture

### Module Dependency Graph

```
                                    ┌─────────────────┐
                                    │   AppModule     │
                                    └────────┬────────┘
                                             │
           ┌─────────────────────────────────┼─────────────────────────────────┐
           │                                 │                                 │
           ▼                                 ▼                                 ▼
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│   ConfigModule      │         │   DatabaseModule    │         │    CacheModule      │
│   (Global)          │         │   - PrismaService   │         │    - RedisService   │
└─────────────────────┘         │   - TimescaleService│         └─────────────────────┘
                                └─────────────────────┘
                                             │
           ┌─────────────────────────────────┼─────────────────────────────────┐
           │                                 │                                 │
           ▼                                 ▼                                 ▼
┌─────────────────────┐         ┌─────────────────────┐         ┌─────────────────────┐
│   AuthModule        │         │   QueueModule       │         │   HealthModule      │
│   - ApiKeyGuard     │         │   - BullMQ          │         │   - HealthChecks    │
│   - ApiKeyStrategy  │         │   - Producers       │         └─────────────────────┘
└─────────────────────┘         │   - Consumers       │
                                └─────────────────────┘
           │
           │  (Protected by AuthGuard)
           │
           ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                              DOMAIN MODULES                                           │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐ │
│   │ Strategies  │   │  Protocols  │   │   Chains    │   │      Benchmarks         │ │
│   │  Module     │   │   Module    │   │   Module    │   │        Module           │ │
│   └──────┬──────┘   └─────────────┘   └─────────────┘   └─────────────────────────┘ │
│          │                                                                           │
│          │ depends on                                                                │
│          ▼                                                                           │
│   ┌─────────────┐   ┌─────────────┐   ┌─────────────┐   ┌─────────────────────────┐ │
│   │   Vaults    │◄──│  Portfolio  │   │Transactions │   │       Webhooks          │ │
│   │   Module    │   │   Module    │   │   Module    │   │        Module           │ │
│   └──────┬──────┘   └──────┬──────┘   └──────┬──────┘   └─────────────────────────┘ │
│          │                 │                 │                                       │
│          └─────────────────┴─────────────────┘                                       │
│                            │                                                         │
│                            ▼                                                         │
│                    ┌─────────────┐                                                   │
│                    │   Users     │                                                   │
│                    │   Module    │                                                   │
│                    └─────────────┘                                                   │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
           │
           │  Uses
           ▼
┌──────────────────────────────────────────────────────────────────────────────────────┐
│                          INFRASTRUCTURE MODULES                                       │
├──────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                       │
│   ┌─────────────────────┐   ┌─────────────────────┐   ┌─────────────────────────┐   │
│   │  BlockchainModule   │   │  ExternalApisModule │   │   NotificationsModule   │   │
│   │  - EthereumProvider │   │  - DefiLlamaService │   │   - EmailService        │   │
│   │  - BaseProvider     │   │  - AavescanService  │   │   - SlackService        │   │
│   │  - MulticallService │   │  - MorphoService    │   └─────────────────────────┘   │
│   └─────────────────────┘   │  - PendleService    │                                  │
│                             │  - LidoService      │                                  │
│                             │  - CoingeckoService │                                  │
│                             └─────────────────────┘                                  │
│                                                                                       │
└──────────────────────────────────────────────────────────────────────────────────────┘
```

---

### Service Layer Pattern

```typescript
// Example: VaultsService following Clean Architecture

// 1. Interface (Port)
interface IVaultsRepository {
  findById(id: string): Promise<Vault | null>;
  findByNeobank(neobankId: string): Promise<Vault[]>;
  create(data: CreateVaultData): Promise<Vault>;
  update(id: string, data: UpdateVaultData): Promise<Vault>;
}

// 2. Repository Implementation (Adapter)
@Injectable()
class VaultsRepository implements IVaultsRepository {
  constructor(private prisma: PrismaService) {}

  async findById(id: string): Promise<Vault | null> {
    return this.prisma.vault.findUnique({ where: { id } });
  }
  // ...
}

// 3. Service (Use Case)
@Injectable()
class VaultsService {
  constructor(
    private readonly vaultsRepository: VaultsRepository,
    private readonly vaultDeploymentService: VaultDeploymentService,
    private readonly cacheService: CacheService,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async createVault(dto: CreateVaultDto, neobank: Neobank): Promise<Vault> {
    // Business logic here
    const vault = await this.vaultsRepository.create({
      ...dto,
      neobankId: neobank.id,
      status: VaultStatus.DEPLOYING,
    });

    // Emit event for async processing
    this.eventEmitter.emit('vault.created', new VaultCreatedEvent(vault));

    return vault;
  }
}

// 4. Controller (Presentation)
@Controller('vaults')
@UseGuards(ApiKeyGuard)
class VaultsController {
  constructor(private readonly vaultsService: VaultsService) {}

  @Post()
  @ApiOperation({ summary: 'Create a new vault' })
  async create(
    @Body() dto: CreateVaultDto,
    @CurrentNeobank() neobank: Neobank,
  ): Promise<VaultResponseDto> {
    const vault = await this.vaultsService.createVault(dto, neobank);
    return VaultResponseDto.fromEntity(vault);
  }
}
```

---

## SDK Architecture

### Client Architecture

```typescript
// packages/sdk/src/client/RagaClient.ts

import { HttpClient } from './HttpClient';
import { StrategiesModule } from '../modules/strategies';
import { VaultsModule } from '../modules/vaults';
import { PortfolioModule } from '../modules/portfolio';
import { TransactionsModule } from '../modules/transactions';

export interface RagaClientConfig {
  apiKey: string;
  baseUrl?: string;
  environment?: 'mainnet' | 'testnet';
  timeout?: number;
  retries?: number;
}

export class RagaClient {
  private readonly httpClient: HttpClient;

  // Module instances (lazy loaded)
  public readonly strategies: StrategiesModule;
  public readonly vaults: VaultsModule;
  public readonly portfolio: PortfolioModule;
  public readonly transactions: TransactionsModule;
  public readonly protocols: ProtocolsModule;
  public readonly chains: ChainsModule;
  public readonly benchmarks: BenchmarksModule;

  constructor(config: RagaClientConfig) {
    this.httpClient = new HttpClient({
      baseUrl: config.baseUrl ?? this.getDefaultBaseUrl(config.environment),
      apiKey: config.apiKey,
      timeout: config.timeout ?? 30000,
      retries: config.retries ?? 3,
    });

    // Initialize modules
    this.strategies = new StrategiesModule(this.httpClient);
    this.vaults = new VaultsModule(this.httpClient);
    this.portfolio = new PortfolioModule(this.httpClient);
    this.transactions = new TransactionsModule(this.httpClient);
    this.protocols = new ProtocolsModule(this.httpClient);
    this.chains = new ChainsModule(this.httpClient);
    this.benchmarks = new BenchmarksModule(this.httpClient);
  }

  private getDefaultBaseUrl(env?: 'mainnet' | 'testnet'): string {
    return env === 'testnet'
      ? 'https://api-testnet.raga.finance/v1'
      : 'https://api.raga.finance/v1';
  }
}
```

### Module Pattern

```typescript
// packages/sdk/src/modules/strategies/strategies.module.ts

import { HttpClient } from '../../client/HttpClient';
import {
  Strategy,
  StrategyDetails,
  GetStrategiesParams,
  GetStrategiesResponse,
  GetStrategyDetailsParams,
} from './strategies.types';

export class StrategiesModule {
  constructor(private readonly http: HttpClient) {}

  /**
   * Get all available yield strategies
   */
  async getAll(params?: GetStrategiesParams): Promise<GetStrategiesResponse> {
    return this.http.get<GetStrategiesResponse>('/strategies', { params });
  }

  /**
   * Get detailed information for specific strategies
   */
  async getDetails(identifiers: string[]): Promise<Record<string, StrategyDetails>> {
    const response = await this.http.post<{ data: Record<string, StrategyDetails> }>(
      '/strategies/details',
      { identifiers }
    );
    return response.data;
  }

  /**
   * Get a single strategy by identifier
   */
  async getById(identifier: string): Promise<StrategyDetails> {
    const details = await this.getDetails([identifier]);
    const strategy = details[identifier];
    if (!strategy) {
      throw new RagaError('STRATEGY_NOT_FOUND', `Strategy ${identifier} not found`);
    }
    return strategy;
  }
}
```

---

## Configuration Management

### Environment Variables

```bash
# .env.example

# Application
NODE_ENV=development
PORT=3000
API_VERSION=v1

# Database
DATABASE_URL=postgresql://user:password@localhost:5432/raga_finance
TIMESCALE_URL=postgresql://user:password@localhost:5433/raga_timescale

# Redis
REDIS_HOST=localhost
REDIS_PORT=6379
REDIS_PASSWORD=

# Authentication
API_KEY_SECRET=your-secret-key-for-hashing

# Blockchain RPC
ETHEREUM_RPC_URL=https://eth-mainnet.g.alchemy.com/v2/your-key
BASE_RPC_URL=https://base-mainnet.g.alchemy.com/v2/your-key
BSC_RPC_URL=https://bsc-dataseed.binance.org
ARBITRUM_RPC_URL=https://arb-mainnet.g.alchemy.com/v2/your-key

# External APIs
DEFILLAMA_API_URL=https://api.llama.fi
AAVESCAN_API_KEY=your-aavescan-key
COINGECKO_API_KEY=your-coingecko-key

# AWS (for production)
AWS_REGION=us-east-1
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=

# Monitoring
SENTRY_DSN=
```

### Configuration Module

```typescript
// apps/api/src/config/configuration.ts

export default () => ({
  app: {
    env: process.env.NODE_ENV || 'development',
    port: parseInt(process.env.PORT, 10) || 3000,
    apiVersion: process.env.API_VERSION || 'v1',
  },
  database: {
    url: process.env.DATABASE_URL,
    timescaleUrl: process.env.TIMESCALE_URL,
  },
  redis: {
    host: process.env.REDIS_HOST || 'localhost',
    port: parseInt(process.env.REDIS_PORT, 10) || 6379,
    password: process.env.REDIS_PASSWORD,
  },
  auth: {
    apiKeySecret: process.env.API_KEY_SECRET,
  },
  blockchain: {
    ethereum: { rpcUrl: process.env.ETHEREUM_RPC_URL },
    base: { rpcUrl: process.env.BASE_RPC_URL },
    bsc: { rpcUrl: process.env.BSC_RPC_URL },
    arbitrum: { rpcUrl: process.env.ARBITRUM_RPC_URL },
  },
  externalApis: {
    defillama: { baseUrl: process.env.DEFILLAMA_API_URL },
    aavescan: { apiKey: process.env.AAVESCAN_API_KEY },
    coingecko: { apiKey: process.env.COINGECKO_API_KEY },
  },
});
```

---

## Dependency Graph

### Build Order (Turborepo)

```json
// turbo.json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": [".env"],
  "pipeline": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".next/**"]
    },
    "lint": {
      "dependsOn": ["^lint"]
    },
    "test": {
      "dependsOn": ["build"]
    },
    "test:e2e": {
      "dependsOn": ["build"]
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "db:generate": {
      "cache": false
    },
    "db:migrate": {
      "cache": false
    }
  }
}
```

### Package Dependencies

```
┌─────────────────────────────────────────────────────────────────┐
│                        BUILD ORDER                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Level 0 (No dependencies)                                     │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │  tsconfig    │  │ eslint-config│  │   config     │         │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
│   Level 1 (Depends on Level 0)                                  │
│   ┌──────────────┐  ┌──────────────┐                            │
│   │    types     │  │  contracts   │                            │
│   └──────────────┘  └──────────────┘                            │
│                                                                  │
│   Level 2 (Depends on Level 1)                                  │
│   ┌──────────────┐                                               │
│   │     sdk      │                                               │
│   └──────────────┘                                               │
│                                                                  │
│   Level 3 (Apps - Depend on packages)                           │
│   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐         │
│   │     api      │  │data-aggregator│  │exec-engine  │         │
│   └──────────────┘  └──────────────┘  └──────────────┘         │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## Next Steps

1. **Initialize Monorepo** - Set up Turborepo with pnpm
2. **Create Package Structure** - Set up shared packages
3. **Bootstrap NestJS API** - Create API app with core modules
4. **Bootstrap SDK** - Create SDK package structure
5. **Set up CI/CD** - GitHub Actions pipelines
6. **Docker Configuration** - Local development environment
