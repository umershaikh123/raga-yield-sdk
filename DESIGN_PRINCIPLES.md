# Raga Finance - Design Principles & Best Practices

## Overview

This document defines the engineering principles, design patterns, and best practices that guide the development of the Raga Finance yield infrastructure platform. These principles ensure code quality, maintainability, security, and scalability.

---

## Table of Contents

1. [Core Principles](#core-principles)
2. [SOLID Principles](#solid-principles)
3. [Clean Architecture](#clean-architecture)
4. [Domain-Driven Design](#domain-driven-design)
5. [API Design Guidelines](#api-design-guidelines)
6. [Error Handling Strategy](#error-handling-strategy)
7. [Security Best Practices](#security-best-practices)
8. [Testing Strategy](#testing-strategy)
9. [Observability & Monitoring](#observability--monitoring)
10. [Code Style & Conventions](#code-style--conventions)
11. [Git Workflow](#git-workflow)
12. [Performance Guidelines](#performance-guidelines)
13. [Database Best Practices](#database-best-practices)

---

## Core Principles

### 1. Simplicity Over Complexity

> "Simple things should be simple, complex things should be possible." — Alan Kay

- Prefer straightforward solutions over clever ones
- Avoid premature optimization
- If code requires extensive comments to explain, it's too complex
- Each function should do one thing well

### 2. Explicit Over Implicit

- Favor explicit dependencies over magic
- Configuration should be visible and documented
- Avoid hidden side effects
- Make intent clear in naming and structure

### 3. Fail Fast, Fail Loud

- Validate inputs at system boundaries
- Throw meaningful errors early
- Never swallow exceptions silently
- Log errors with full context

### 4. Composition Over Inheritance

- Prefer interfaces and composition
- Use dependency injection
- Keep inheritance hierarchies shallow (max 2-3 levels)
- Favor small, focused modules

### 5. Convention Over Configuration

- Follow established patterns consistently
- Use sensible defaults
- Document deviations from conventions
- Automate repetitive decisions

---

## SOLID Principles

### S - Single Responsibility Principle

Each class/module should have one reason to change.

```typescript
// ❌ BAD: Multiple responsibilities
class VaultService {
  async createVault(dto: CreateVaultDto) { /* ... */ }
  async deployContract(vault: Vault) { /* ... */ }
  async sendNotification(vault: Vault) { /* ... */ }
  async calculateAPY(vault: Vault) { /* ... */ }
}

// ✅ GOOD: Single responsibility
class VaultService {
  constructor(
    private readonly vaultRepository: VaultRepository,
    private readonly deploymentService: VaultDeploymentService,
    private readonly notificationService: NotificationService,
  ) {}

  async createVault(dto: CreateVaultDto): Promise<Vault> {
    const vault = await this.vaultRepository.create(dto);
    await this.deploymentService.queueDeployment(vault);
    await this.notificationService.notifyVaultCreated(vault);
    return vault;
  }
}
```

### O - Open/Closed Principle

Open for extension, closed for modification.

```typescript
// ✅ GOOD: Strategy pattern for protocol adapters
interface IProtocolAdapter {
  getAPY(poolAddress: string): Promise<number>;
  getTVL(poolAddress: string): Promise<bigint>;
}

class AaveAdapter implements IProtocolAdapter {
  async getAPY(poolAddress: string): Promise<number> { /* Aave-specific */ }
  async getTVL(poolAddress: string): Promise<bigint> { /* Aave-specific */ }
}

class MorphoAdapter implements IProtocolAdapter {
  async getAPY(poolAddress: string): Promise<number> { /* Morpho-specific */ }
  async getTVL(poolAddress: string): Promise<bigint> { /* Morpho-specific */ }
}

// Adding new protocol = new adapter, no changes to existing code
class PendleAdapter implements IProtocolAdapter { /* ... */ }
```

### L - Liskov Substitution Principle

Subtypes must be substitutable for their base types.

```typescript
// ✅ GOOD: All adapters work interchangeably
class ProtocolDataService {
  constructor(private readonly adapters: Map<string, IProtocolAdapter>) {}

  async getAPY(protocol: string, poolAddress: string): Promise<number> {
    const adapter = this.adapters.get(protocol);
    if (!adapter) throw new Error(`Unknown protocol: ${protocol}`);
    return adapter.getAPY(poolAddress); // Works with any adapter
  }
}
```

### I - Interface Segregation Principle

Clients should not depend on interfaces they don't use.

```typescript
// ❌ BAD: Fat interface
interface IVaultOperations {
  deposit(amount: bigint): Promise<void>;
  withdraw(shares: bigint): Promise<void>;
  rebalance(): Promise<void>;
  claimRewards(): Promise<void>;
  setFees(fee: number): Promise<void>;
  pause(): Promise<void>;
  unpause(): Promise<void>;
}

// ✅ GOOD: Segregated interfaces
interface IVaultUser {
  deposit(amount: bigint): Promise<void>;
  withdraw(shares: bigint): Promise<void>;
}

interface IVaultAdmin {
  setFees(fee: number): Promise<void>;
  pause(): Promise<void>;
  unpause(): Promise<void>;
}

interface IVaultExecutor {
  rebalance(): Promise<void>;
  claimRewards(): Promise<void>;
}
```

### D - Dependency Inversion Principle

Depend on abstractions, not concretions.

```typescript
// ❌ BAD: Direct dependency on implementation
class VaultService {
  private prisma = new PrismaClient(); // Concrete dependency

  async findVault(id: string) {
    return this.prisma.vault.findUnique({ where: { id } });
  }
}

// ✅ GOOD: Depend on abstraction
interface IVaultRepository {
  findById(id: string): Promise<Vault | null>;
  create(data: CreateVaultData): Promise<Vault>;
}

@Injectable()
class VaultService {
  constructor(
    @Inject('IVaultRepository')
    private readonly vaultRepository: IVaultRepository,
  ) {}

  async findVault(id: string): Promise<Vault | null> {
    return this.vaultRepository.findById(id);
  }
}
```

---

## Clean Architecture

### Layer Separation

```
┌─────────────────────────────────────────────────────────────────┐
│                     PRESENTATION LAYER                          │
│  Controllers, DTOs, Validators, Serializers                    │
│  - Handles HTTP requests/responses                              │
│  - Input validation                                             │
│  - Response formatting                                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     APPLICATION LAYER                           │
│  Services, Use Cases, Event Handlers                           │
│  - Business logic orchestration                                 │
│  - Transaction management                                       │
│  - Event emission                                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                       DOMAIN LAYER                              │
│  Entities, Value Objects, Domain Services, Interfaces          │
│  - Core business rules                                          │
│  - No external dependencies                                     │
│  - Framework agnostic                                           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   INFRASTRUCTURE LAYER                          │
│  Repositories, External APIs, Database, Cache, Queue           │
│  - Implementations of interfaces                                │
│  - External service clients                                     │
│  - Framework-specific code                                      │
└─────────────────────────────────────────────────────────────────┘
```

### Dependency Rule

Dependencies only point inward:
- Infrastructure depends on Domain
- Application depends on Domain
- Presentation depends on Application and Domain
- Domain depends on nothing

```typescript
// Domain Layer (no dependencies)
// entities/vault.entity.ts
export class Vault {
  constructor(
    public readonly id: string,
    public readonly name: string,
    public readonly chainId: number,
    public readonly allocations: VaultAllocation[],
    private _status: VaultStatus,
  ) {}

  get status(): VaultStatus {
    return this._status;
  }

  activate(): void {
    if (this._status !== VaultStatus.DEPLOYING) {
      throw new DomainError('Cannot activate vault in current state');
    }
    this._status = VaultStatus.ACTIVE;
  }

  calculateBlendedAPY(): number {
    return this.allocations.reduce(
      (sum, alloc) => sum + alloc.apy * (alloc.percentage / 100),
      0,
    );
  }
}

// Application Layer (depends on Domain)
// services/vaults.service.ts
@Injectable()
export class VaultsService {
  constructor(
    private readonly vaultRepository: IVaultRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  async activateVault(vaultId: string): Promise<Vault> {
    const vault = await this.vaultRepository.findById(vaultId);
    if (!vault) throw new NotFoundException('Vault not found');

    vault.activate(); // Domain logic

    await this.vaultRepository.save(vault);
    this.eventEmitter.emit('vault.activated', vault);

    return vault;
  }
}

// Infrastructure Layer (implements Domain interfaces)
// repositories/vaults.repository.ts
@Injectable()
export class PrismaVaultRepository implements IVaultRepository {
  constructor(private readonly prisma: PrismaService) {}

  async findById(id: string): Promise<Vault | null> {
    const data = await this.prisma.vault.findUnique({
      where: { id },
      include: { allocations: true },
    });

    return data ? this.toDomain(data) : null;
  }

  private toDomain(data: PrismaVault): Vault {
    return new Vault(
      data.id,
      data.name,
      data.chainId,
      data.allocations.map(a => new VaultAllocation(a)),
      data.status as VaultStatus,
    );
  }
}

// Presentation Layer (depends on Application)
// controllers/vaults.controller.ts
@Controller('vaults')
export class VaultsController {
  constructor(private readonly vaultsService: VaultsService) {}

  @Post(':id/activate')
  async activate(@Param('id') id: string): Promise<VaultResponseDto> {
    const vault = await this.vaultsService.activateVault(id);
    return VaultResponseDto.fromEntity(vault);
  }
}
```

---

## Domain-Driven Design

### Bounded Contexts

```
┌─────────────────────────────────────────────────────────────────┐
│                    RAGA FINANCE DOMAIN                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │   STRATEGY       │  │     VAULT        │  │  PORTFOLIO   │  │
│  │   CONTEXT        │  │    CONTEXT       │  │   CONTEXT    │  │
│  │                  │  │                  │  │              │  │
│  │  - Strategy      │  │  - Vault         │  │  - Position  │  │
│  │  - Protocol      │  │  - Allocation    │  │  - PnL       │  │
│  │  - Chain         │  │  - Fee           │  │  - Snapshot  │  │
│  │  - Token         │  │  - Curator       │  │              │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────┐  │
│  │   TRANSACTION    │  │     USER         │  │  ANALYTICS   │  │
│  │   CONTEXT        │  │    CONTEXT       │  │   CONTEXT    │  │
│  │                  │  │                  │  │              │  │
│  │  - Deposit       │  │  - User          │  │  - Benchmark │  │
│  │  - Withdrawal    │  │  - Neobank       │  │  - Risk      │  │
│  │  - TxPayload     │  │  - ApiKey        │  │  - Metric    │  │
│  └──────────────────┘  └──────────────────┘  └──────────────┘  │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Aggregates

```typescript
// Vault is an Aggregate Root
// All modifications go through the root

class Vault {
  private readonly allocations: VaultAllocation[] = [];

  addAllocation(strategyId: string, percentage: number): void {
    const totalPercentage = this.allocations.reduce(
      (sum, a) => sum + a.percentage,
      0,
    );

    if (totalPercentage + percentage > 100) {
      throw new DomainError('Total allocation cannot exceed 100%');
    }

    this.allocations.push(new VaultAllocation(strategyId, percentage));
  }

  removeAllocation(strategyId: string): void {
    const index = this.allocations.findIndex(a => a.strategyId === strategyId);
    if (index === -1) {
      throw new DomainError('Allocation not found');
    }
    this.allocations.splice(index, 1);
  }
}
```

### Value Objects

```typescript
// Value Objects are immutable and compared by value

class Money {
  constructor(
    public readonly amount: bigint,
    public readonly currency: string,
    public readonly decimals: number,
  ) {
    if (amount < 0n) {
      throw new DomainError('Amount cannot be negative');
    }
  }

  add(other: Money): Money {
    if (this.currency !== other.currency) {
      throw new DomainError('Cannot add different currencies');
    }
    return new Money(this.amount + other.amount, this.currency, this.decimals);
  }

  toFormatted(): string {
    const divisor = 10n ** BigInt(this.decimals);
    const whole = this.amount / divisor;
    const fraction = this.amount % divisor;
    return `${whole}.${fraction.toString().padStart(this.decimals, '0')}`;
  }

  equals(other: Money): boolean {
    return (
      this.amount === other.amount &&
      this.currency === other.currency
    );
  }
}

class APY {
  constructor(public readonly basisPoints: number) {
    if (basisPoints < 0 || basisPoints > 100000) {
      throw new DomainError('APY must be between 0% and 1000%');
    }
  }

  toPercentage(): number {
    return this.basisPoints / 100;
  }

  static fromPercentage(percentage: number): APY {
    return new APY(Math.round(percentage * 100));
  }
}
```

---

## API Design Guidelines

### RESTful Conventions

```
# Resource Naming
GET    /v1/strategies              # List strategies
POST   /v1/strategies/details      # Get details for multiple (batch operation)
GET    /v1/strategies/:id          # Get single strategy

GET    /v1/vaults                  # List vaults
POST   /v1/vaults                  # Create vault
GET    /v1/vaults/:id              # Get vault
PUT    /v1/vaults/:id              # Update vault
DELETE /v1/vaults/:id              # Delete vault (if supported)

# Nested Resources
GET    /v1/vaults/:id/allocations  # List vault allocations
GET    /v1/users/:id/portfolio     # Get user portfolio
GET    /v1/users/:id/transactions  # Get user transactions

# Actions (RPC-style for complex operations)
POST   /v1/transactions/deposit/prepare    # Prepare deposit
POST   /v1/transactions/simulate           # Simulate transaction
POST   /v1/vaults/:id/rebalance            # Trigger rebalance
```

### Request/Response Standards

```typescript
// Standard Success Response
interface ApiResponse<T> {
  success: true;
  data: T;
  meta: {
    timestamp: string;
    requestId: string;
  };
}

// Paginated Response
interface PaginatedResponse<T> extends ApiResponse<T[]> {
  pagination: {
    page: number;
    pageSize: number;
    totalItems: number;
    totalPages: number;
    hasNext: boolean;
    hasPrev: boolean;
  };
}

// Error Response
interface ErrorResponse {
  success: false;
  error: {
    code: string;
    message: string;
    details?: Record<string, unknown>;
  };
  meta: {
    timestamp: string;
    requestId: string;
  };
}
```

### Versioning Strategy

```typescript
// URL-based versioning (preferred for breaking changes)
// /v1/strategies
// /v2/strategies

// Header-based versioning (for minor variations)
// X-API-Version: 2024-01-15

@Controller('v1/strategies')
export class StrategiesV1Controller { }

@Controller('v2/strategies')
export class StrategiesV2Controller { }
```

### Input Validation

```typescript
// Use class-validator with strict validation

class CreateVaultDto {
  @IsString()
  @MinLength(3)
  @MaxLength(100)
  name: string;

  @IsInt()
  @IsIn([1, 8453, 56, 42161])
  chainId: number;

  @IsString()
  @IsEthereumAddress()
  depositToken: string;

  @IsObject()
  @ValidateNested()
  @Type(() => AllocationDto)
  allocations: Record<string, number>;

  @IsNumber()
  @Min(0)
  @Max(10)
  @IsOptional()
  feePercentage?: number;
}

class AllocationDto {
  @IsNumber()
  @Min(0)
  @Max(100)
  percentage: number;
}
```

---

## Error Handling Strategy

### Error Hierarchy

```typescript
// Base error class
export class AppError extends Error {
  constructor(
    public readonly code: string,
    message: string,
    public readonly statusCode: number = 500,
    public readonly details?: Record<string, unknown>,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

// Domain errors (business rule violations)
export class DomainError extends AppError {
  constructor(message: string, details?: Record<string, unknown>) {
    super('DOMAIN_ERROR', message, 400, details);
  }
}

// Application errors
export class NotFoundError extends AppError {
  constructor(resource: string, id: string) {
    super('NOT_FOUND', `${resource} with id ${id} not found`, 404, { resource, id });
  }
}

export class ValidationError extends AppError {
  constructor(errors: ValidationErrorDetail[]) {
    super('VALIDATION_ERROR', 'Validation failed', 400, { errors });
  }
}

export class UnauthorizedError extends AppError {
  constructor(message = 'Unauthorized') {
    super('UNAUTHORIZED', message, 401);
  }
}

export class ForbiddenError extends AppError {
  constructor(message = 'Forbidden') {
    super('FORBIDDEN', message, 403);
  }
}

// Infrastructure errors
export class ExternalServiceError extends AppError {
  constructor(service: string, message: string) {
    super('EXTERNAL_SERVICE_ERROR', message, 502, { service });
  }
}

export class DatabaseError extends AppError {
  constructor(message: string) {
    super('DATABASE_ERROR', message, 500);
  }
}
```

### Global Exception Filter

```typescript
@Catch()
export class AllExceptionsFilter implements ExceptionFilter {
  constructor(
    private readonly logger: Logger,
    private readonly configService: ConfigService,
  ) {}

  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const response = ctx.getResponse<Response>();
    const request = ctx.getRequest<Request>();

    const { status, errorResponse } = this.buildErrorResponse(exception, request);

    // Log error with context
    this.logger.error({
      message: errorResponse.error.message,
      code: errorResponse.error.code,
      requestId: request.id,
      path: request.url,
      method: request.method,
      stack: exception instanceof Error ? exception.stack : undefined,
    });

    // Don't expose internal errors in production
    if (this.configService.get('app.env') === 'production' && status >= 500) {
      errorResponse.error.message = 'Internal server error';
      delete errorResponse.error.details;
    }

    response.status(status).json(errorResponse);
  }

  private buildErrorResponse(exception: unknown, request: Request) {
    if (exception instanceof AppError) {
      return {
        status: exception.statusCode,
        errorResponse: {
          success: false,
          error: {
            code: exception.code,
            message: exception.message,
            details: exception.details,
          },
          meta: {
            timestamp: new Date().toISOString(),
            requestId: request.id,
          },
        },
      };
    }

    // Handle NestJS HTTP exceptions
    if (exception instanceof HttpException) {
      return {
        status: exception.getStatus(),
        errorResponse: {
          success: false,
          error: {
            code: 'HTTP_ERROR',
            message: exception.message,
          },
          meta: {
            timestamp: new Date().toISOString(),
            requestId: request.id,
          },
        },
      };
    }

    // Unknown errors
    return {
      status: 500,
      errorResponse: {
        success: false,
        error: {
          code: 'INTERNAL_ERROR',
          message: 'An unexpected error occurred',
        },
        meta: {
          timestamp: new Date().toISOString(),
          requestId: request.id,
        },
      },
    };
  }
}
```

---

## Security Best Practices

### 1. Authentication & Authorization

```typescript
// API Key validation
@Injectable()
export class ApiKeyGuard implements CanActivate {
  constructor(
    private readonly apiKeyService: ApiKeyService,
    private readonly cacheService: CacheService,
  ) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'];

    if (!apiKey) {
      throw new UnauthorizedError('API key is required');
    }

    // Check cache first
    const cached = await this.cacheService.get(`apikey:${apiKey}`);
    if (cached) {
      request.neobank = JSON.parse(cached);
      return true;
    }

    // Validate API key
    const neobank = await this.apiKeyService.validateKey(apiKey);
    if (!neobank) {
      throw new UnauthorizedError('Invalid API key');
    }

    // Cache for 5 minutes
    await this.cacheService.set(`apikey:${apiKey}`, JSON.stringify(neobank), 300);
    request.neobank = neobank;

    return true;
  }
}
```

### 2. Rate Limiting

```typescript
// Per-key rate limiting
@Injectable()
export class RateLimitGuard implements CanActivate {
  constructor(private readonly redisService: RedisService) {}

  async canActivate(context: ExecutionContext): Promise<boolean> {
    const request = context.switchToHttp().getRequest();
    const apiKey = request.headers['x-api-key'] || request.ip;

    const key = `ratelimit:${apiKey}`;
    const limit = 300; // requests per minute
    const window = 60; // seconds

    const current = await this.redisService.incr(key);

    if (current === 1) {
      await this.redisService.expire(key, window);
    }

    if (current > limit) {
      throw new HttpException('Rate limit exceeded', HttpStatus.TOO_MANY_REQUESTS);
    }

    // Set headers
    const response = context.switchToHttp().getResponse();
    response.setHeader('X-RateLimit-Limit', limit);
    response.setHeader('X-RateLimit-Remaining', Math.max(0, limit - current));

    return true;
  }
}
```

### 3. Input Sanitization

```typescript
// Sanitize all string inputs
class SanitizePipe implements PipeTransform {
  transform(value: unknown) {
    if (typeof value === 'string') {
      return this.sanitize(value);
    }
    if (typeof value === 'object' && value !== null) {
      return this.sanitizeObject(value);
    }
    return value;
  }

  private sanitize(str: string): string {
    return str
      .trim()
      .replace(/[<>]/g, '') // Basic XSS prevention
      .slice(0, 10000); // Length limit
  }

  private sanitizeObject(obj: Record<string, unknown>): Record<string, unknown> {
    const result: Record<string, unknown> = {};
    for (const [key, value] of Object.entries(obj)) {
      result[this.sanitize(key)] = this.transform(value);
    }
    return result;
  }
}
```

### 4. Secrets Management

```typescript
// Never log secrets
const sensitiveKeys = ['apiKey', 'password', 'secret', 'token', 'privateKey'];

function redactSensitive(obj: Record<string, unknown>): Record<string, unknown> {
  const result: Record<string, unknown> = {};

  for (const [key, value] of Object.entries(obj)) {
    if (sensitiveKeys.some(k => key.toLowerCase().includes(k))) {
      result[key] = '[REDACTED]';
    } else if (typeof value === 'object' && value !== null) {
      result[key] = redactSensitive(value as Record<string, unknown>);
    } else {
      result[key] = value;
    }
  }

  return result;
}
```

### 5. Database Security

```sql
-- Use parameterized queries (Prisma handles this)
-- Never interpolate user input into SQL

-- Principle of least privilege
CREATE ROLE api_service LOGIN PASSWORD 'xxx';
GRANT SELECT, INSERT, UPDATE ON strategies TO api_service;
GRANT SELECT, INSERT, UPDATE, DELETE ON vaults TO api_service;
-- No DROP, TRUNCATE, ALTER permissions
```

---

## Testing Strategy

### Testing Pyramid

```
                    ┌───────────────┐
                    │     E2E       │  10%
                    │    Tests      │  Slow, expensive
                    └───────┬───────┘
                            │
                    ┌───────┴───────┐
                    │  Integration  │  20%
                    │    Tests      │  Medium speed
                    └───────┬───────┘
                            │
            ┌───────────────┴───────────────┐
            │          Unit Tests           │  70%
            │         Fast, isolated         │  Most coverage
            └───────────────────────────────┘
```

### Unit Tests

```typescript
// Test business logic in isolation

describe('Vault', () => {
  describe('addAllocation', () => {
    it('should add allocation when total is under 100%', () => {
      const vault = createVault();
      vault.addAllocation('strategy-1', 50);

      expect(vault.getAllocations()).toHaveLength(1);
      expect(vault.getAllocations()[0].percentage).toBe(50);
    });

    it('should throw when total exceeds 100%', () => {
      const vault = createVault();
      vault.addAllocation('strategy-1', 60);

      expect(() => vault.addAllocation('strategy-2', 50)).toThrow(
        'Total allocation cannot exceed 100%',
      );
    });
  });

  describe('calculateBlendedAPY', () => {
    it('should calculate weighted average APY', () => {
      const vault = createVaultWithAllocations([
        { strategyId: 's1', percentage: 50, apy: 10 },
        { strategyId: 's2', percentage: 50, apy: 20 },
      ]);

      expect(vault.calculateBlendedAPY()).toBe(15);
    });
  });
});
```

### Integration Tests

```typescript
// Test module integration with real database

describe('VaultsService (Integration)', () => {
  let service: VaultsService;
  let prisma: PrismaService;

  beforeAll(async () => {
    const module = await Test.createTestingModule({
      imports: [DatabaseModule, VaultsModule],
    }).compile();

    service = module.get<VaultsService>(VaultsService);
    prisma = module.get<PrismaService>(PrismaService);
  });

  beforeEach(async () => {
    await prisma.cleanDatabase();
  });

  it('should create vault with allocations', async () => {
    const neobank = await createTestNeobank(prisma);
    const dto: CreateVaultDto = {
      name: 'Test Vault',
      chainId: 1,
      depositToken: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
      allocations: { 'strategy-1': 100 },
    };

    const vault = await service.createVault(dto, neobank);

    expect(vault.name).toBe('Test Vault');
    expect(vault.status).toBe(VaultStatus.DEPLOYING);

    const saved = await prisma.vault.findUnique({
      where: { id: vault.id },
      include: { allocations: true },
    });
    expect(saved?.allocations).toHaveLength(1);
  });
});
```

### E2E Tests

```typescript
// Test full API flow

describe('Vaults API (E2E)', () => {
  let app: INestApplication;
  let apiKey: string;

  beforeAll(async () => {
    app = await createTestApp();
    apiKey = await createTestApiKey(app);
  });

  describe('POST /v1/vaults', () => {
    it('should create vault', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/vaults')
        .set('X-API-Key', apiKey)
        .send({
          name: 'Test Vault',
          chainId: 1,
          depositToken: '0xA0b86991c6218b36c1d19D4a2e9Eb0cE3606eB48',
          allocations: { 'aave-v3-usdc-ethereum': 100 },
        });

      expect(response.status).toBe(201);
      expect(response.body.success).toBe(true);
      expect(response.body.data.id).toBeDefined();
      expect(response.body.data.status).toBe('deploying');
    });

    it('should reject invalid chain', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/vaults')
        .set('X-API-Key', apiKey)
        .send({
          name: 'Test Vault',
          chainId: 999,
          depositToken: '0x...',
          allocations: {},
        });

      expect(response.status).toBe(400);
      expect(response.body.error.code).toBe('VALIDATION_ERROR');
    });

    it('should reject without API key', async () => {
      const response = await request(app.getHttpServer())
        .post('/v1/vaults')
        .send({});

      expect(response.status).toBe(401);
    });
  });
});
```

---

## Observability & Monitoring

### Structured Logging

```typescript
// Use structured JSON logs

@Injectable()
export class LoggerService {
  private readonly logger: pino.Logger;

  constructor(private readonly config: ConfigService) {
    this.logger = pino({
      level: config.get('LOG_LEVEL') || 'info',
      formatters: {
        level: (label) => ({ level: label }),
      },
      base: {
        service: 'raga-api',
        env: config.get('NODE_ENV'),
      },
    });
  }

  info(message: string, context?: Record<string, unknown>) {
    this.logger.info(context, message);
  }

  error(message: string, error?: Error, context?: Record<string, unknown>) {
    this.logger.error(
      {
        ...context,
        error: error ? {
          message: error.message,
          stack: error.stack,
          name: error.name,
        } : undefined,
      },
      message,
    );
  }
}

// Log output
// {"level":"info","service":"raga-api","env":"production","requestId":"abc123","method":"POST","path":"/v1/vaults","message":"Vault created"}
```

### Request Tracing

```typescript
// Add request ID to all logs

@Injectable()
export class RequestIdMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const requestId = req.headers['x-request-id'] || uuidv4();
    req['id'] = requestId;
    res.setHeader('X-Request-Id', requestId);

    // Add to async local storage for logging
    asyncLocalStorage.run({ requestId }, () => next());
  }
}
```

### Health Checks

```typescript
@Controller('health')
export class HealthController {
  constructor(
    private readonly health: HealthCheckService,
    private readonly db: PrismaHealthIndicator,
    private readonly redis: RedisHealthIndicator,
  ) {}

  @Get()
  @HealthCheck()
  check() {
    return this.health.check([
      () => this.db.pingCheck('database'),
      () => this.redis.pingCheck('redis'),
    ]);
  }

  @Get('ready')
  ready() {
    return { status: 'ok', timestamp: new Date().toISOString() };
  }

  @Get('live')
  live() {
    return { status: 'ok' };
  }
}
```

### Metrics

```typescript
// Prometheus metrics

import { Counter, Histogram, Registry } from 'prom-client';

const register = new Registry();

const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status'],
  registers: [register],
});

const httpRequestTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total number of HTTP requests',
  labelNames: ['method', 'route', 'status'],
  registers: [register],
});

// Middleware to collect metrics
@Injectable()
export class MetricsMiddleware implements NestMiddleware {
  use(req: Request, res: Response, next: NextFunction) {
    const start = Date.now();

    res.on('finish', () => {
      const duration = (Date.now() - start) / 1000;
      const labels = {
        method: req.method,
        route: req.route?.path || 'unknown',
        status: res.statusCode.toString(),
      };

      httpRequestDuration.observe(labels, duration);
      httpRequestTotal.inc(labels);
    });

    next();
  }
}
```

---

## Code Style & Conventions

### Naming Conventions

```typescript
// Files
user.service.ts          // Services
user.controller.ts       // Controllers
user.repository.ts       // Repositories
user.entity.ts           // Entities
user.dto.ts              // DTOs
user.interface.ts        // Interfaces
user.spec.ts             // Tests

// Classes
class UserService { }           // PascalCase
class CreateUserDto { }         // PascalCase with suffix
interface IUserRepository { }   // I prefix for interfaces (optional)

// Variables and functions
const userName = 'John';        // camelCase
function getUserById() { }      // camelCase

// Constants
const MAX_RETRY_COUNT = 3;      // SCREAMING_SNAKE_CASE
const API_VERSION = 'v1';

// Enums
enum VaultStatus {
  DEPLOYING = 'deploying',      // PascalCase key, lowercase value
  ACTIVE = 'active',
  PAUSED = 'paused',
}

// Booleans
const isActive = true;          // is/has/can/should prefix
const hasPermission = false;
const canEdit = true;
```

### Code Organization

```typescript
// Imports order
// 1. Node.js built-ins
import { readFile } from 'fs/promises';

// 2. External packages
import { Injectable } from '@nestjs/common';
import { PrismaClient } from '@prisma/client';

// 3. Internal packages (monorepo)
import { Strategy } from '@raga-finance/types';

// 4. Relative imports
import { VaultRepository } from './vault.repository';
import { CreateVaultDto } from './dto/create-vault.dto';

// Class structure order
@Injectable()
export class VaultService {
  // 1. Private properties
  private readonly cache = new Map();

  // 2. Constructor
  constructor(
    private readonly vaultRepository: VaultRepository,
    private readonly eventEmitter: EventEmitter2,
  ) {}

  // 3. Public methods
  async createVault(dto: CreateVaultDto): Promise<Vault> { }

  async getVault(id: string): Promise<Vault> { }

  // 4. Private methods
  private validateAllocations(allocations: Allocation[]): void { }
}
```

---

## Git Workflow

### Branch Naming

```
main                    # Production branch
develop                 # Development branch
feature/ABC-123-add-vault-api     # Feature branches
bugfix/ABC-456-fix-apy-calculation
hotfix/ABC-789-security-patch
release/v1.2.0          # Release branches
```

### Commit Messages

```
# Format: <type>(<scope>): <description>

feat(vaults): add vault creation endpoint
fix(portfolio): correct PnL calculation for partial withdrawals
docs(api): update OpenAPI specification
refactor(strategies): extract normalization logic to service
test(vaults): add integration tests for allocation
chore(deps): update NestJS to v10.3
perf(cache): optimize strategy caching strategy

# Types:
# feat     - New feature
# fix      - Bug fix
# docs     - Documentation
# refactor - Code refactoring
# test     - Adding tests
# chore    - Maintenance
# perf     - Performance improvement
# ci       - CI/CD changes
```

### Pull Request Template

```markdown
## Description
Brief description of changes

## Type of Change
- [ ] Bug fix
- [ ] New feature
- [ ] Breaking change
- [ ] Documentation update

## Testing
- [ ] Unit tests added/updated
- [ ] Integration tests added/updated
- [ ] E2E tests added/updated
- [ ] Manual testing completed

## Checklist
- [ ] Code follows project style guidelines
- [ ] Self-review completed
- [ ] Documentation updated
- [ ] No new warnings
- [ ] Dependent changes merged
```

---

## Performance Guidelines

### Caching Strategy

```typescript
// Multi-layer caching

// Layer 1: In-memory (for frequently accessed, rarely changed data)
const PROTOCOL_CACHE = new Map<string, Protocol>();

// Layer 2: Redis (for shared state across instances)
await redis.setex(`strategy:${id}`, 300, JSON.stringify(strategy));

// Layer 3: Database (source of truth)
const strategy = await prisma.strategy.findUnique({ where: { id } });

// Cache-aside pattern
async getStrategy(id: string): Promise<Strategy> {
  // Check Redis
  const cached = await this.redis.get(`strategy:${id}`);
  if (cached) return JSON.parse(cached);

  // Get from DB
  const strategy = await this.repository.findById(id);
  if (!strategy) throw new NotFoundError('Strategy', id);

  // Cache for 5 minutes
  await this.redis.setex(`strategy:${id}`, 300, JSON.stringify(strategy));

  return strategy;
}
```

### Database Query Optimization

```typescript
// Use proper indexes
// Include only needed fields
// Avoid N+1 queries

// ❌ BAD: N+1 query
const vaults = await prisma.vault.findMany();
for (const vault of vaults) {
  vault.allocations = await prisma.allocation.findMany({
    where: { vaultId: vault.id },
  });
}

// ✅ GOOD: Single query with include
const vaults = await prisma.vault.findMany({
  include: {
    allocations: {
      include: { strategy: true },
    },
  },
});

// ✅ GOOD: Select only needed fields
const vaults = await prisma.vault.findMany({
  select: {
    id: true,
    name: true,
    currentApy: true,
    tvlUsd: true,
  },
});
```

### Pagination

```typescript
// Cursor-based pagination for large datasets

interface PaginationParams {
  cursor?: string;
  limit?: number;
}

async getStrategies(params: PaginationParams) {
  const limit = Math.min(params.limit || 20, 100);

  const strategies = await prisma.strategy.findMany({
    take: limit + 1, // Get one extra to check hasNext
    cursor: params.cursor ? { id: params.cursor } : undefined,
    skip: params.cursor ? 1 : 0,
    orderBy: { tvlUsd: 'desc' },
  });

  const hasNext = strategies.length > limit;
  if (hasNext) strategies.pop();

  return {
    data: strategies,
    cursor: strategies.length > 0 ? strategies[strategies.length - 1].id : null,
    hasNext,
  };
}
```

---

## Database Best Practices

### Migrations

```typescript
// Use Prisma migrations for schema changes
// npx prisma migrate dev --name add_vault_status

// Always make migrations reversible
// Test migrations on staging before production
// Never modify existing migrations
```

### Indexes

```sql
-- Index for common queries
CREATE INDEX idx_strategies_active ON strategies(is_active) WHERE is_active = true;
CREATE INDEX idx_vaults_neobank_status ON vaults(neobank_id, status);
CREATE INDEX idx_transactions_user_type ON transactions(user_id, type, created_at DESC);

-- Composite indexes for multi-column queries
CREATE INDEX idx_strategies_chain_type_apy ON strategies(chain_id, strategy_type, total_apy DESC);
```

### Connection Pooling

```typescript
// Configure Prisma connection pool

// prisma/schema.prisma
datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

// Connection URL with pool config
// DATABASE_URL="postgresql://user:pass@host:5432/db?connection_limit=20&pool_timeout=10"
```

---

## Summary

Following these principles ensures:

1. **Maintainability** - Code is easy to understand and modify
2. **Testability** - Components can be tested in isolation
3. **Scalability** - System can grow without major rewrites
4. **Security** - Best practices prevent common vulnerabilities
5. **Performance** - Efficient use of resources
6. **Reliability** - Proper error handling and monitoring
7. **Consistency** - Uniform code style across the team

These principles should be enforced through:
- Code reviews
- Automated linting (ESLint)
- CI/CD pipelines
- Team discussions and documentation
