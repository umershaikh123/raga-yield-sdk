# Raga Finance - Project Context

## What We're Building

**Yield Infrastructure Layer for Web3 Neobanks** - A platform that enables neobanks to offer DeFi investment products to their users through a unified SDK and API.

## Core Goals

1. **For Neobanks**: Discover yield opportunities, create vaults, manage strategies, view analytics
2. **For End Users**: Invest in vaults, withdraw anytime, track PnL
3. **For Raga**: Protocol-agnostic, non-custodial, scalable infrastructure

## Tech Stack

| Component | Technology |
|-----------|------------|
| Monorepo | Turborepo + pnpm |
| API | NestJS + TypeScript |
| Database | PostgreSQL + TimescaleDB |
| Cache/Queue | Redis Streams |
| SDK | TypeScript |
| Data Scrapers | Python |
| Execution Engine | Go (TEE) |
| Deployment | AWS ECS Fargate |

## Project Structure

```
raga-finance/
├── apps/
│   ├── api/                 # NestJS REST API
│   ├── data-aggregator/     # Python protocol scrapers
│   ├── execution-engine/    # Go fund allocation (TEE)
│   └── vault-deployer/      # Vault deployment bot
├── packages/
│   ├── sdk/                 # @raga-finance/sdk
│   ├── types/               # @raga-finance/types
│   ├── contracts/           # ABIs & contract types
│   └── config/              # Shared configs
└── infrastructure/          # Terraform, Docker
```

## Supported Protocols (v1)

Aave, Morpho, Pendle, Euler, Spectra, Lido

## Supported Chains (v1)

Ethereum (1), Base (8453), BSC (56), Arbitrum (42161)

## Key API Endpoints

| Endpoint | Purpose |
|----------|---------|
| `GET /v1/strategies` | List yield strategies |
| `POST /v1/strategies/details` | Get strategy details |
| `POST /v1/vaults` | Create vault |
| `GET /v1/vaults` | List neobank vaults |
| `GET /v1/users/:id/portfolio` | User portfolio |
| `POST /v1/transactions/deposit/prepare` | Prepare deposit tx |

## SDK Usage

```typescript
import { RagaClient } from '@raga-finance/sdk';

const raga = new RagaClient({ apiKey: 'xxx' });

// Fetch strategies
const strategies = await raga.strategies.getAll({ chainId: 1 });

// Create vault
const vault = await raga.vaults.create({
  chainId: 1,
  name: 'USD Vault',
  allocations: { 'aave-usdc': 100 }
});

// Get portfolio
const portfolio = await raga.portfolio.get('user-123');
```

## Documentation

- [ARCHITECTURE.md](./ARCHITECTURE.md) - Full system architecture
- [PROJECT_STRUCTURE.md](./PROJECT_STRUCTURE.md) - Detailed folder structure
- [DESIGN_PRINCIPLES.md](./DESIGN_PRINCIPLES.md) - Engineering best practices
- [API_SPECIFICATION.md](./API_SPECIFICATION.md) - Complete API contracts

## Design Principles

- Clean Architecture with DDD
- SOLID principles
- Repository pattern
- Dependency injection
- Comprehensive error handling
- Type-safe throughout
