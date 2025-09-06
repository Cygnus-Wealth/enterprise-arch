# Solana Integration Domain

## Domain Overview

The Solana Integration domain provides read-only access to Solana blockchain data using Domain-Driven Design principles. It offers high-performance balance fetching for SOL, SPL tokens, and NFTs with enterprise-grade resilience patterns including circuit breakers, caching, and connection pooling.

## Core Responsibilities

### Primary Functions
1. **Balance Fetching**: SOL native balance and SPL token balances
2. **NFT Discovery**: Enumerate and fetch NFT metadata
3. **Transaction History**: Retrieve and parse transaction data
4. **Program Interactions**: Decode program-specific data
5. **Real-time Updates**: Event-driven architecture for live data
6. **Performance Optimization**: LRU caching and connection pooling

### Domain Boundaries
- **Owns**: Solana RPC interactions, data transformation, caching logic
- **Does NOT Own**: Wallet connections, transaction signing, key management

## Domain-Driven Architecture

### Bounded Contexts
```
sol-integration/
├── domain/
│   ├── aggregates/       # Domain aggregates
│   ├── entities/         # Domain entities
│   ├── value-objects/    # Immutable value objects
│   └── events/          # Domain events
├── application/
│   ├── services/        # Application services
│   ├── facades/         # Public API facades
│   └── dto/            # Data transfer objects
├── infrastructure/
│   ├── rpc/            # RPC client implementation
│   ├── cache/          # Caching layer
│   └── mappers/        # Data mappers
└── interfaces/
    └── api/            # External API contracts
```

### Core Aggregates
```typescript
// Portfolio Aggregate Root
class SolanaPortfolio {
  private readonly address: SolanaAddress
  private balances: Map<TokenMint, TokenBalance>
  private nfts: NFTCollection
  private lastUpdated: Date
  
  // Business logic
  getTotalValue(prices: PriceMap): Money
  hasToken(mint: TokenMint): boolean
  getTokenBalance(mint: TokenMint): TokenBalance | null
  
  // Domain events
  onBalanceUpdated(event: BalanceUpdatedEvent): void
  onNFTReceived(event: NFTReceivedEvent): void
}

// Token Balance Value Object
class TokenBalance {
  constructor(
    readonly amount: bigint,
    readonly decimals: number,
    readonly mint: TokenMint
  ) {}
  
  toFormatted(): string
  toNumber(): number
  equals(other: TokenBalance): boolean
}
```

## Service Layer

### Facade Pattern
```typescript
class SolanaIntegrationFacade {
  // High-level API for external consumers
  async getPortfolioSnapshot(address: string): Promise<PortfolioSnapshot>
  async getTokenBalances(address: string): Promise<TokenBalance[]>
  async getNFTCollection(address: string): Promise<NFT[]>
  async getTransactionHistory(address: string, limit?: number): Promise<Transaction[]>
  
  // Real-time subscriptions
  subscribeToBalance(address: string): Observable<BalanceUpdate>
  subscribeToTransactions(address: string): Observable<Transaction>
}
```

### Application Services
```typescript
class BalanceFetcherService {
  constructor(
    private rpcClient: SolanaRpcClient,
    private cache: CacheService,
    private circuitBreaker: CircuitBreaker
  ) {}
  
  async fetchBalance(address: PublicKey): Promise<Balance> {
    return this.circuitBreaker.execute(async () => {
      // Check cache first
      const cached = await this.cache.get(address)
      if (cached) return cached
      
      // Fetch from RPC
      const balance = await this.rpcClient.getBalance(address)
      
      // Cache result
      await this.cache.set(address, balance, TTL)
      
      return balance
    })
  }
}
```

## Infrastructure Layer

### RPC Client
```typescript
class SolanaRpcClient {
  private connectionPool: ConnectionPool
  private readonly config: RpcConfig
  
  constructor(config: RpcConfig) {
    this.config = config
    this.connectionPool = new ConnectionPool({
      endpoints: config.endpoints,
      maxConnections: config.maxConnections,
      strategy: 'round-robin'
    })
  }
  
  async getBalance(address: PublicKey): Promise<number> {
    const connection = await this.connectionPool.getConnection()
    return connection.getBalance(address)
  }
}

interface RpcConfig {
  endpoints: RpcEndpoint[]
  maxConnections: number
  requestTimeout: number
  commitment: Commitment
}
```

### Connection Pool
```typescript
class ConnectionPool {
  private connections: Map<string, Connection>
  private healthChecker: HealthChecker
  
  async getConnection(): Promise<Connection> {
    // Get healthy connection using strategy
    const endpoint = await this.selectEndpoint()
    return this.connections.get(endpoint.url)
  }
  
  private async selectEndpoint(): Promise<RpcEndpoint> {
    // Round-robin, least-connections, or weighted
    const healthy = await this.healthChecker.getHealthyEndpoints()
    return this.strategy.select(healthy)
  }
}
```

## Resilience Patterns

### Circuit Breaker
```typescript
class CircuitBreaker {
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN' = 'CLOSED'
  private failures: number = 0
  private lastFailureTime?: Date
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    if (this.state === 'OPEN') {
      if (this.shouldAttemptReset()) {
        this.state = 'HALF_OPEN'
      } else {
        throw new CircuitOpenError()
      }
    }
    
    try {
      const result = await operation()
      this.onSuccess()
      return result
    } catch (error) {
      this.onFailure()
      throw error
    }
  }
  
  private onSuccess(): void {
    this.failures = 0
    this.state = 'CLOSED'
  }
  
  private onFailure(): void {
    this.failures++
    this.lastFailureTime = new Date()
    
    if (this.failures >= this.threshold) {
      this.state = 'OPEN'
    }
  }
}
```

### Retry Logic
```typescript
class RetryPolicy {
  constructor(
    private maxAttempts: number = 3,
    private backoffStrategy: BackoffStrategy
  ) {}
  
  async execute<T>(operation: () => Promise<T>): Promise<T> {
    let lastError: Error
    
    for (let attempt = 0; attempt < this.maxAttempts; attempt++) {
      try {
        return await operation()
      } catch (error) {
        lastError = error
        
        if (!this.isRetryable(error)) {
          throw error
        }
        
        const delay = this.backoffStrategy.getDelay(attempt)
        await this.sleep(delay)
      }
    }
    
    throw lastError
  }
}
```

## Caching Strategy

### LRU Cache Implementation
```typescript
class LRUCache<K, V> {
  private cache: Map<K, CacheEntry<V>>
  private readonly maxSize: number
  private readonly ttl: number
  
  constructor(maxSize: number, ttl: number) {
    this.cache = new Map()
    this.maxSize = maxSize
    this.ttl = ttl
  }
  
  get(key: K): V | null {
    const entry = this.cache.get(key)
    
    if (!entry) return null
    
    if (this.isExpired(entry)) {
      this.cache.delete(key)
      return null
    }
    
    // Move to front (most recently used)
    this.cache.delete(key)
    this.cache.set(key, entry)
    
    return entry.value
  }
  
  set(key: K, value: V): void {
    // Remove oldest if at capacity
    if (this.cache.size >= this.maxSize) {
      const firstKey = this.cache.keys().next().value
      this.cache.delete(firstKey)
    }
    
    this.cache.set(key, {
      value,
      timestamp: Date.now()
    })
  }
}
```

### Cache Configuration
```typescript
interface CacheConfig {
  // Cache sizes
  balanceCacheSize: 10000
  tokenCacheSize: 50000
  nftCacheSize: 5000
  
  // TTL values (ms)
  balanceTTL: 60_000      // 1 minute
  tokenTTL: 300_000       // 5 minutes
  nftTTL: 3600_000        // 1 hour
  
  // Cache warming
  warmupOnStartup: boolean
  warmupAddresses: string[]
}
```

## Data Models

### Domain Entities
```typescript
// SPL Token Entity
class SPLToken {
  constructor(
    readonly mint: PublicKey,
    readonly owner: PublicKey,
    readonly amount: bigint,
    readonly decimals: number,
    readonly tokenAccount: PublicKey
  ) {}
  
  getFormattedBalance(): string {
    return (Number(this.amount) / Math.pow(10, this.decimals)).toString()
  }
}

// NFT Entity
class SolanaNFT {
  constructor(
    readonly mint: PublicKey,
    readonly owner: PublicKey,
    readonly metadata: NFTMetadata,
    readonly collection?: PublicKey
  ) {}
  
  isPartOfCollection(collectionMint: PublicKey): boolean {
    return this.collection?.equals(collectionMint) ?? false
  }
}

interface NFTMetadata {
  name: string
  symbol: string
  uri: string
  sellerFeeBasisPoints: number
  creators: Creator[]
  collection?: Collection
  attributes?: Attribute[]
}
```

### Value Objects
```typescript
// Solana Address Value Object
class SolanaAddress {
  private readonly value: PublicKey
  
  constructor(address: string | PublicKey) {
    this.value = typeof address === 'string' 
      ? new PublicKey(address)
      : address
  }
  
  toString(): string {
    return this.value.toBase58()
  }
  
  equals(other: SolanaAddress): boolean {
    return this.value.equals(other.value)
  }
  
  static isValid(address: string): boolean {
    try {
      new PublicKey(address)
      return true
    } catch {
      return false
    }
  }
}

// Lamports Value Object
class Lamports {
  constructor(readonly value: bigint) {}
  
  toSOL(): number {
    return Number(this.value) / LAMPORTS_PER_SOL
  }
  
  static fromSOL(sol: number): Lamports {
    return new Lamports(BigInt(sol * LAMPORTS_PER_SOL))
  }
}
```

## Event-Driven Architecture

### Domain Events
```typescript
abstract class DomainEvent {
  readonly occurredAt: Date = new Date()
  abstract readonly aggregateId: string
  abstract readonly eventType: string
}

class BalanceUpdatedEvent extends DomainEvent {
  constructor(
    readonly aggregateId: string,
    readonly address: string,
    readonly oldBalance: bigint,
    readonly newBalance: bigint,
    readonly tokenMint?: string
  ) {
    super()
  }
  
  readonly eventType = 'BALANCE_UPDATED'
}

class NFTReceivedEvent extends DomainEvent {
  constructor(
    readonly aggregateId: string,
    readonly nftMint: string,
    readonly from: string,
    readonly to: string
  ) {
    super()
  }
  
  readonly eventType = 'NFT_RECEIVED'
}
```

### Event Bus
```typescript
class EventBus {
  private handlers: Map<string, EventHandler[]> = new Map()
  
  subscribe(eventType: string, handler: EventHandler): void {
    const handlers = this.handlers.get(eventType) || []
    handlers.push(handler)
    this.handlers.set(eventType, handlers)
  }
  
  async publish(event: DomainEvent): Promise<void> {
    const handlers = this.handlers.get(event.eventType) || []
    
    await Promise.all(
      handlers.map(handler => handler.handle(event))
    )
  }
}
```

## Performance Metrics

### Monitoring
```typescript
interface PerformanceMetrics {
  // Response times
  avgBalanceFetchTime: number
  avgTokenFetchTime: number
  avgNFTFetchTime: number
  
  // Cache metrics
  cacheHitRate: number
  cacheSize: number
  cacheEvictions: number
  
  // RPC metrics
  rpcRequestCount: number
  rpcErrorRate: number
  rpcLatency: Histogram
  
  // Circuit breaker metrics
  circuitBreakerState: string
  circuitBreakerFailures: number
}

class MetricsCollector {
  private metrics: PerformanceMetrics
  
  recordLatency(operation: string, duration: number): void
  recordCacheHit(cacheType: string): void
  recordCacheMiss(cacheType: string): void
  recordRpcError(error: Error): void
  
  getMetrics(): PerformanceMetrics
  reset(): void
}
```

## Error Handling

### Error Types
```typescript
class SolanaIntegrationError extends Error {
  constructor(
    message: string,
    readonly code: ErrorCode,
    readonly details?: any
  ) {
    super(message)
  }
}

enum ErrorCode {
  RPC_ERROR = 'RPC_ERROR',
  INVALID_ADDRESS = 'INVALID_ADDRESS',
  NETWORK_ERROR = 'NETWORK_ERROR',
  TIMEOUT = 'TIMEOUT',
  RATE_LIMIT = 'RATE_LIMIT',
  INSUFFICIENT_BALANCE = 'INSUFFICIENT_BALANCE',
  ACCOUNT_NOT_FOUND = 'ACCOUNT_NOT_FOUND'
}
```

## Testing Strategy

### Unit Tests
```typescript
describe('SolanaPortfolio Aggregate', () => {
  test('calculates total value correctly', () => {
    const portfolio = new SolanaPortfolio(mockAddress)
    portfolio.addBalance(mockTokenBalance)
    
    const totalValue = portfolio.getTotalValue(mockPrices)
    expect(totalValue.amount).toBe(expectedValue)
  })
})
```

### Integration Tests
```typescript
describe('Balance Fetcher Service', () => {
  test('fetches balance with circuit breaker', async () => {
    const service = new BalanceFetcherService(
      mockRpcClient,
      mockCache,
      mockCircuitBreaker
    )
    
    const balance = await service.fetchBalance(testAddress)
    expect(balance).toBeDefined()
  })
})
```

## Configuration

### Domain Configuration
```typescript
interface SolanaConfig {
  // RPC Configuration
  rpcEndpoints: string[]
  commitment: 'processed' | 'confirmed' | 'finalized'
  maxConcurrentRequests: number
  
  // Cache Configuration
  enableCache: boolean
  cacheConfig: CacheConfig
  
  // Resilience Configuration
  circuitBreakerThreshold: number
  circuitBreakerTimeout: number
  retryAttempts: number
  retryBackoff: 'exponential' | 'linear'
  
  // Performance
  connectionPoolSize: number
  requestTimeout: number
}
```

## Future Enhancements

### Planned Features
1. **Staking Accounts**: Support for stake account discovery
2. **DeFi Integrations**: Decode DEX and lending positions
3. **Transaction Streaming**: Real-time transaction monitoring
4. **Program Analytics**: Analyze program interactions
5. **Compressed NFTs**: Support for compressed NFT standards
6. **Priority Fees**: Intelligent priority fee calculation