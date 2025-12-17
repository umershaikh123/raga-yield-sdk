# SDK Development Guide

## Overview

This guide outlines the requirements and tasks for developing the `@raga-finance/sdk` TypeScript library.

---

## SDK Goals

1. **Simple Integration**: Neobanks can integrate yield features with minimal code
2. **Type-Safe**: Full TypeScript support with comprehensive types
3. **Platform Agnostic**: Works in Node.js and browsers
4. **Developer Experience**: Intuitive API, excellent documentation

---

## Tech Stack

| Tool | Purpose |
|------|---------|
| TypeScript | Language |
| tsup | Build/bundle (ESM + CJS) |
| Vitest | Testing |
| Zod | Runtime validation |
| Axios/Fetch | HTTP client |
| Changesets | Versioning |

---

## Project Structure

```
packages/sdk/
├── src/
│   ├── index.ts                    # Public exports
│   │
│   ├── client/
│   │   ├── RagaClient.ts           # Main client class
│   │   ├── HttpClient.ts           # HTTP wrapper with retry
│   │   ├── errors.ts               # SDK error classes
│   │   └── types.ts                # Client configuration types
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
│   │   ├── formatters.ts           # APY, currency, number formatters
│   │   ├── validators.ts           # Input validation helpers
│   │   ├── retry.ts                # Retry with exponential backoff
│   │   ├── cache.ts                # In-memory response cache
│   │   └── constants.ts            # Chain IDs, URLs, etc.
│   │
│   └── types/
│       └── index.ts                # Re-export from @raga-finance/types
│
├── tests/
│   ├── unit/
│   │   ├── client.test.ts
│   │   ├── http-client.test.ts
│   │   └── modules/
│   │       ├── strategies.test.ts
│   │       ├── vaults.test.ts
│   │       └── portfolio.test.ts
│   │
│   ├── integration/
│   │   └── api.integration.test.ts
│   │
│   └── mocks/
│       ├── handlers.ts             # MSW handlers
│       └── fixtures.ts             # Test data
│
├── examples/
│   ├── basic-usage.ts
│   ├── fetch-strategies.ts
│   ├── create-vault.ts
│   ├── portfolio-tracking.ts
│   └── prepare-transaction.ts
│
├── package.json
├── tsconfig.json
├── tsup.config.ts
├── vitest.config.ts
└── README.md
```

---

## Development Tasks

### Phase 1: Project Setup

- [ ] **Task 1.1**: Initialize package with pnpm
  ```bash
  cd packages/sdk
  pnpm init
  ```

- [ ] **Task 1.2**: Configure TypeScript
  ```json
  // tsconfig.json
  {
    "extends": "../tsconfig/sdk.json",
    "compilerOptions": {
      "outDir": "dist",
      "rootDir": "src"
    },
    "include": ["src/**/*"]
  }
  ```

- [ ] **Task 1.3**: Configure tsup for dual ESM/CJS builds
  ```typescript
  // tsup.config.ts
  import { defineConfig } from 'tsup';

  export default defineConfig({
    entry: ['src/index.ts'],
    format: ['cjs', 'esm'],
    dts: true,
    splitting: false,
    sourcemap: true,
    clean: true,
  });
  ```

- [ ] **Task 1.4**: Configure Vitest for testing
  ```typescript
  // vitest.config.ts
  import { defineConfig } from 'vitest/config';

  export default defineConfig({
    test: {
      globals: true,
      environment: 'node',
      coverage: {
        reporter: ['text', 'json', 'html'],
      },
    },
  });
  ```

- [ ] **Task 1.5**: Set up package.json exports
  ```json
  {
    "name": "@raga-finance/sdk",
    "version": "0.1.0",
    "main": "./dist/index.cjs",
    "module": "./dist/index.js",
    "types": "./dist/index.d.ts",
    "exports": {
      ".": {
        "import": "./dist/index.js",
        "require": "./dist/index.cjs",
        "types": "./dist/index.d.ts"
      }
    },
    "files": ["dist"],
    "scripts": {
      "build": "tsup",
      "dev": "tsup --watch",
      "test": "vitest",
      "test:coverage": "vitest --coverage",
      "lint": "eslint src/",
      "typecheck": "tsc --noEmit"
    }
  }
  ```

---

### Phase 2: Core Client Implementation

- [ ] **Task 2.1**: Create error classes
  ```typescript
  // src/client/errors.ts
  export class RagaError extends Error {
    constructor(
      public code: string,
      message: string,
      public details?: Record<string, unknown>,
    ) {
      super(message);
      this.name = 'RagaError';
    }
  }

  export class RagaApiError extends RagaError {
    constructor(
      code: string,
      message: string,
      public statusCode: number,
      details?: Record<string, unknown>,
    ) {
      super(code, message, details);
      this.name = 'RagaApiError';
    }
  }

  export class RagaNetworkError extends RagaError {
    constructor(message: string) {
      super('NETWORK_ERROR', message);
      this.name = 'RagaNetworkError';
    }
  }

  export class RagaValidationError extends RagaError {
    constructor(message: string, details?: Record<string, unknown>) {
      super('VALIDATION_ERROR', message, details);
      this.name = 'RagaValidationError';
    }
  }
  ```

- [ ] **Task 2.2**: Create HTTP client with retry logic
  ```typescript
  // src/client/HttpClient.ts
  export interface HttpClientConfig {
    baseUrl: string;
    apiKey: string;
    timeout?: number;
    retries?: number;
  }

  export class HttpClient {
    private readonly config: Required<HttpClientConfig>;

    constructor(config: HttpClientConfig) {
      this.config = {
        timeout: 30000,
        retries: 3,
        ...config,
      };
    }

    async get<T>(path: string, options?: RequestOptions): Promise<T> { }
    async post<T>(path: string, body?: unknown, options?: RequestOptions): Promise<T> { }
    async put<T>(path: string, body?: unknown, options?: RequestOptions): Promise<T> { }
    async delete<T>(path: string, options?: RequestOptions): Promise<T> { }

    private async request<T>(method: string, path: string, options?: RequestOptions): Promise<T> {
      // Implement with retry logic, error handling
    }
  }
  ```

- [ ] **Task 2.3**: Create main RagaClient class
  ```typescript
  // src/client/RagaClient.ts
  export interface RagaClientConfig {
    apiKey: string;
    baseUrl?: string;
    environment?: 'mainnet' | 'testnet';
    timeout?: number;
    retries?: number;
  }

  export class RagaClient {
    public readonly strategies: StrategiesModule;
    public readonly protocols: ProtocolsModule;
    public readonly chains: ChainsModule;
    public readonly vaults: VaultsModule;
    public readonly portfolio: PortfolioModule;
    public readonly transactions: TransactionsModule;
    public readonly benchmarks: BenchmarksModule;

    constructor(config: RagaClientConfig) {
      const httpClient = new HttpClient({
        baseUrl: config.baseUrl ?? this.getDefaultUrl(config.environment),
        apiKey: config.apiKey,
        timeout: config.timeout,
        retries: config.retries,
      });

      this.strategies = new StrategiesModule(httpClient);
      this.protocols = new ProtocolsModule(httpClient);
      this.chains = new ChainsModule(httpClient);
      this.vaults = new VaultsModule(httpClient);
      this.portfolio = new PortfolioModule(httpClient);
      this.transactions = new TransactionsModule(httpClient);
      this.benchmarks = new BenchmarksModule(httpClient);
    }

    private getDefaultUrl(env?: 'mainnet' | 'testnet'): string {
      return env === 'testnet'
        ? 'https://api-testnet.raga.finance/v1'
        : 'https://api.raga.finance/v1';
    }
  }
  ```

---

### Phase 3: Module Implementation

#### Task 3.1: Strategies Module

```typescript
// src/modules/strategies/strategies.types.ts
export interface Strategy {
  id: string;
  identifier: string;
  name: string;
  protocol: {
    id: string;
    name: string;
    logo: string;
  };
  chain: {
    id: number;
    name: string;
  };
  type: 'lending' | 'staking' | 'lp' | 'vault';
  depositToken: {
    address: string;
    symbol: string;
    decimals: number;
  };
  metrics: {
    apy: number;
    rewardApy: number;
    totalApy: number;
    tvlUsd: number;
  };
  risk: {
    score: number;
    profile: 'low' | 'medium' | 'high';
  };
}

export interface GetStrategiesParams {
  chainId?: number;
  chainIds?: number[];
  protocol?: string;
  type?: string;
  minApy?: number;
  maxApy?: number;
  minTvl?: number;
  riskProfile?: 'low' | 'medium' | 'high';
  sortBy?: 'apy' | 'tvl' | 'risk_score';
  sortOrder?: 'asc' | 'desc';
  page?: number;
  pageSize?: number;
}

export interface StrategyDetails extends Strategy {
  description: string;
  contractAddress: string;
  charts: {
    apy: Record<string, number>;
    tvl: Record<string, number>;
  };
}
```

```typescript
// src/modules/strategies/strategies.module.ts
export class StrategiesModule {
  constructor(private readonly http: HttpClient) {}

  async getAll(params?: GetStrategiesParams): Promise<PaginatedResponse<Strategy>> {
    return this.http.get('/strategies', { params });
  }

  async getDetails(identifiers: string[]): Promise<Record<string, StrategyDetails>> {
    const response = await this.http.post<ApiResponse<Record<string, StrategyDetails>>>(
      '/strategies/details',
      { identifiers }
    );
    return response.data;
  }

  async getById(identifier: string): Promise<StrategyDetails> {
    const details = await this.getDetails([identifier]);
    if (!details[identifier]) {
      throw new RagaError('NOT_FOUND', `Strategy ${identifier} not found`);
    }
    return details[identifier];
  }
}
```

#### Task 3.2: Vaults Module

```typescript
// src/modules/vaults/vaults.module.ts
export class VaultsModule {
  constructor(private readonly http: HttpClient) {}

  async create(params: CreateVaultParams): Promise<Vault> {
    const response = await this.http.post<ApiResponse<Vault>>('/vaults', params);
    return response.data;
  }

  async list(params?: ListVaultsParams): Promise<VaultsListResponse> {
    return this.http.get('/vaults', { params });
  }

  async get(vaultId: string): Promise<VaultDetails> {
    const response = await this.http.get<ApiResponse<VaultDetails>>(`/vaults/${vaultId}`);
    return response.data;
  }

  async update(vaultId: string, allocations: Record<string, number>): Promise<Vault> {
    const response = await this.http.put<ApiResponse<Vault>>(
      `/vaults/${vaultId}`,
      { allocations }
    );
    return response.data;
  }

  async getMetrics(vaultId: string, period?: string): Promise<VaultMetrics> {
    return this.http.get(`/vaults/${vaultId}/metrics`, { params: { period } });
  }
}
```

#### Task 3.3: Portfolio Module

```typescript
// src/modules/portfolio/portfolio.module.ts
export class PortfolioModule {
  constructor(private readonly http: HttpClient) {}

  async get(userId: string): Promise<Portfolio> {
    const response = await this.http.get<ApiResponse<Portfolio>>(
      `/users/${userId}/portfolio`
    );
    return response.data;
  }

  async getPositions(userId: string, vaultId?: string): Promise<Position[]> {
    const response = await this.http.get<ApiResponse<{ positions: Position[] }>>(
      `/users/${userId}/positions`,
      { params: { vault_id: vaultId } }
    );
    return response.data.positions;
  }

  async getTransactions(userId: string, params?: TransactionParams): Promise<PaginatedResponse<Transaction>> {
    return this.http.get(`/users/${userId}/transactions`, { params });
  }

  async getPnl(userId: string, period?: string): Promise<PnlData> {
    const response = await this.http.get<ApiResponse<PnlData>>(
      `/users/${userId}/pnl`,
      { params: { period } }
    );
    return response.data;
  }
}
```

#### Task 3.4: Transactions Module

```typescript
// src/modules/transactions/transactions.module.ts
export class TransactionsModule {
  constructor(private readonly http: HttpClient) {}

  async prepareDeposit(params: PrepareDepositParams): Promise<PreparedTransaction> {
    const response = await this.http.post<ApiResponse<PreparedTransaction>>(
      '/transactions/deposit/prepare',
      params
    );
    return response.data;
  }

  async prepareWithdraw(params: PrepareWithdrawParams): Promise<PreparedTransaction> {
    const response = await this.http.post<ApiResponse<PreparedTransaction>>(
      '/transactions/withdraw/prepare',
      params
    );
    return response.data;
  }

  async simulate(tx: SimulateParams): Promise<SimulationResult> {
    const response = await this.http.post<ApiResponse<SimulationResult>>(
      '/transactions/simulate',
      tx
    );
    return response.data;
  }

  async getStatus(txHash: string): Promise<TransactionStatus> {
    const response = await this.http.get<ApiResponse<TransactionStatus>>(
      `/transactions/${txHash}/status`
    );
    return response.data;
  }

  async waitForConfirmation(
    txHash: string,
    options?: { timeout?: number; interval?: number }
  ): Promise<TransactionStatus> {
    const timeout = options?.timeout ?? 120000;
    const interval = options?.interval ?? 3000;
    const startTime = Date.now();

    while (Date.now() - startTime < timeout) {
      const status = await this.getStatus(txHash);
      if (status.status === 'confirmed' || status.status === 'failed') {
        return status;
      }
      await new Promise(resolve => setTimeout(resolve, interval));
    }

    throw new RagaError('TIMEOUT', 'Transaction confirmation timeout');
  }
}
```

---

### Phase 4: Utilities

- [ ] **Task 4.1**: Create formatters
  ```typescript
  // src/utils/formatters.ts
  export function formatApy(apy: number, decimals = 2): string {
    return `${apy.toFixed(decimals)}%`;
  }

  export function formatUsd(amount: number, decimals = 2): string {
    return new Intl.NumberFormat('en-US', {
      style: 'currency',
      currency: 'USD',
      minimumFractionDigits: decimals,
    }).format(amount);
  }

  export function formatTokenAmount(
    amount: bigint | string,
    decimals: number,
    displayDecimals = 4
  ): string {
    const value = typeof amount === 'string' ? BigInt(amount) : amount;
    const divisor = 10n ** BigInt(decimals);
    const whole = value / divisor;
    const fraction = value % divisor;
    return `${whole}.${fraction.toString().padStart(decimals, '0').slice(0, displayDecimals)}`;
  }

  export function shortenAddress(address: string, chars = 4): string {
    return `${address.slice(0, chars + 2)}...${address.slice(-chars)}`;
  }
  ```

- [ ] **Task 4.2**: Create retry utility
  ```typescript
  // src/utils/retry.ts
  export async function withRetry<T>(
    fn: () => Promise<T>,
    options: { retries: number; delay: number; backoff?: number }
  ): Promise<T> {
    let lastError: Error;
    let delay = options.delay;

    for (let attempt = 0; attempt <= options.retries; attempt++) {
      try {
        return await fn();
      } catch (error) {
        lastError = error as Error;
        if (attempt < options.retries) {
          await new Promise(resolve => setTimeout(resolve, delay));
          delay *= options.backoff ?? 2;
        }
      }
    }

    throw lastError!;
  }
  ```

- [ ] **Task 4.3**: Create cache utility
  ```typescript
  // src/utils/cache.ts
  export class MemoryCache {
    private cache = new Map<string, { value: unknown; expires: number }>();

    get<T>(key: string): T | undefined {
      const entry = this.cache.get(key);
      if (!entry) return undefined;
      if (Date.now() > entry.expires) {
        this.cache.delete(key);
        return undefined;
      }
      return entry.value as T;
    }

    set<T>(key: string, value: T, ttlMs: number): void {
      this.cache.set(key, { value, expires: Date.now() + ttlMs });
    }

    delete(key: string): void {
      this.cache.delete(key);
    }

    clear(): void {
      this.cache.clear();
    }
  }
  ```

---

### Phase 5: Testing

- [ ] **Task 5.1**: Set up MSW for API mocking
  ```typescript
  // tests/mocks/handlers.ts
  import { http, HttpResponse } from 'msw';

  export const handlers = [
    http.get('*/v1/strategies', () => {
      return HttpResponse.json({
        success: true,
        data: [/* mock strategies */],
        pagination: { page: 1, pageSize: 20, totalItems: 100 }
      });
    }),

    http.post('*/v1/vaults', async ({ request }) => {
      const body = await request.json();
      return HttpResponse.json({
        success: true,
        data: {
          id: 'vault_123',
          status: 'deploying',
          ...body
        }
      });
    }),
  ];
  ```

- [ ] **Task 5.2**: Write unit tests for each module
- [ ] **Task 5.3**: Write integration tests against mock server
- [ ] **Task 5.4**: Achieve >80% code coverage

---

### Phase 6: Documentation & Examples

- [ ] **Task 6.1**: Write comprehensive README.md
- [ ] **Task 6.2**: Add JSDoc comments to all public methods
- [ ] **Task 6.3**: Create example files for common use cases
- [ ] **Task 6.4**: Generate API documentation with TypeDoc

---

### Phase 7: Publishing

- [ ] **Task 7.1**: Set up Changesets for versioning
- [ ] **Task 7.2**: Configure GitHub Actions for publishing
- [ ] **Task 7.3**: Publish to npm as `@raga-finance/sdk`

---

## API Reference

### RagaClient

```typescript
const raga = new RagaClient({
  apiKey: 'your-api-key',      // Required
  environment: 'mainnet',       // 'mainnet' | 'testnet'
  baseUrl: 'custom-url',        // Override base URL
  timeout: 30000,               // Request timeout (ms)
  retries: 3,                   // Retry attempts
});
```

### Modules

| Module | Methods |
|--------|---------|
| `raga.strategies` | `getAll()`, `getDetails()`, `getById()` |
| `raga.protocols` | `list()` |
| `raga.chains` | `list()` |
| `raga.vaults` | `create()`, `list()`, `get()`, `update()`, `getMetrics()` |
| `raga.portfolio` | `get()`, `getPositions()`, `getTransactions()`, `getPnl()` |
| `raga.transactions` | `prepareDeposit()`, `prepareWithdraw()`, `simulate()`, `getStatus()`, `waitForConfirmation()` |
| `raga.benchmarks` | `get()` |

---

## Acceptance Criteria

### Functional Requirements

- [ ] All API endpoints are wrapped with typed methods
- [ ] Errors are properly typed and include API error details
- [ ] Retry logic handles transient failures
- [ ] Works in both Node.js and browser environments
- [ ] TypeScript types are exported for all entities

### Non-Functional Requirements

- [ ] Bundle size < 50KB (gzipped)
- [ ] Zero runtime dependencies (except fetch polyfill if needed)
- [ ] Tree-shakeable exports
- [ ] Source maps included
- [ ] 80%+ test coverage

### Documentation

- [ ] README with quick start guide
- [ ] All public methods have JSDoc
- [ ] Examples for common use cases
- [ ] TypeDoc-generated API reference

---

## Timeline Estimate

| Phase | Tasks |
|-------|-------|
| Phase 1 | Project Setup |
| Phase 2 | Core Client |
| Phase 3 | All Modules |
| Phase 4 | Utilities |
| Phase 5 | Testing |
| Phase 6 | Documentation |
| Phase 7 | Publishing |

---

## Dependencies

### Production
```json
{
  "zod": "^3.22.0"  // Optional: runtime validation
}
```

### Development
```json
{
  "typescript": "^5.3.0",
  "tsup": "^8.0.0",
  "vitest": "^1.0.0",
  "msw": "^2.0.0",
  "@types/node": "^20.0.0",
  "typedoc": "^0.25.0"
}
```
