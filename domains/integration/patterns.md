# Integration Patterns Guide

> **Navigation**: [Integration Domain](./README.md) > Integration Patterns

## Overview

This guide provides detailed implementation patterns for System Architects building bounded contexts within the Integration Domain. These patterns ensure consistency, reliability, and maintainability across all integration implementations.

**For context**: This document is referenced from the [Integration Domain README](./README.md), which provides strategic overview and coordination guidance.

## Anti-Corruption Layer Pattern

### Purpose
Protect the domain model from external API changes and inconsistencies while maintaining a stable interface for domain consumers.

### Implementation Structure

```
External API → Adapter → Translator → Validator → Domain Model
```

### Pattern Components

#### 1. Adapter Layer
Handles raw communication with external services:
- HTTP client configuration
- Request/response handling
- Authentication management
- Error code mapping

#### 2. Translator Layer
Transforms external data formats to internal representations:
- Field mapping and renaming
- Type conversion (strings to numbers, dates)
- Unit normalization (wei to ether, cents to dollars)
- Structure flattening or nesting

#### 3. Validator Layer
Ensures data integrity before entering domain:
- Schema validation against expected structure
- Business rule validation (non-negative balances)
- Referential integrity checks
- Sanitization of untrusted data

### Example Structure

```
// Adapter: Raw API interaction
class EVMRPCAdapter {
  async getBalance(address: string): Promise<RawBalanceResponse>
}

// Translator: Format conversion
class EVMDataTranslator {
  translateBalance(raw: RawBalanceResponse): ExternalBalance
}

// Validator: Domain protection
class EVMDataValidator {
  validateBalance(balance: ExternalBalance): DomainBalance
}

// Facade: Clean domain interface
class EVMIntegrationService {
  async getBalance(address: string): Promise<DomainBalance> {
    const raw = await adapter.getBalance(address)
    const external = translator.translateBalance(raw)
    return validator.validateBalance(external)
  }
}
```

## Circuit Breaker Pattern

### Purpose
Prevent cascading failures by failing fast when external services are unavailable or performing poorly.

### States and Transitions

```
CLOSED → (failures > threshold) → OPEN
OPEN → (timeout expires) → HALF_OPEN
HALF_OPEN → (success) → CLOSED
HALF_OPEN → (failure) → OPEN
```

### Configuration Parameters

#### Thresholds
- **Failure Threshold**: 5 failures in 60 seconds
- **Success Threshold**: 3 successes to close
- **Timeout Duration**: 30 seconds in OPEN state
- **Volume Threshold**: Minimum 10 requests for statistics

#### Implementation Guidelines

```typescript
interface CircuitBreakerConfig {
  failureThreshold: number      // Failures before opening
  successThreshold: number      // Successes to close
  timeout: number               // Milliseconds in OPEN state
  volumeThreshold: number       // Minimum requests for stats
  errorFilter?: (error: Error) => boolean  // Which errors count
}

class CircuitBreaker {
  // Track rolling window of outcomes
  private outcomes: RingBuffer<Outcome>

  // Current state management
  private state: 'CLOSED' | 'OPEN' | 'HALF_OPEN'

  // Automatic state transitions
  private scheduleHalfOpen(): void
  private evaluateHealth(): void
}
```

### Fallback Strategies

1. **Return Cached Data**: Use last known good value
2. **Return Default**: Provide safe default response
3. **Throw Specific Error**: Allow upstream handling
4. **Delegate to Backup**: Try alternative service

## Retry Pattern with Exponential Backoff

### Purpose
Handle transient failures gracefully without overwhelming failing services.

### Implementation Formula

```
delay = min(baseDelay * (2 ^ attempt) + jitter, maxDelay)
jitter = random(0, baseDelay * 0.3)
```

### Configuration

```typescript
interface RetryConfig {
  maxAttempts: 3,
  baseDelay: 1000,      // 1 second
  maxDelay: 30000,      // 30 seconds
  multiplier: 2,
  jitterFactor: 0.3,
  retryableErrors: [
    'ECONNRESET',
    'ETIMEDOUT',
    'ENOTFOUND',
    429  // Rate limited
  ]
}
```

### Retry Decision Logic

```typescript
function shouldRetry(error: Error, attempt: number): boolean {
  // Don't retry non-retryable errors
  if (!isRetryable(error)) return false

  // Check attempt limit
  if (attempt >= config.maxAttempts) return false

  // Check circuit breaker state
  if (circuitBreaker.isOpen()) return false

  return true
}
```

## Request Batching and Coalescing

### Purpose
Optimize API usage by combining multiple requests into single calls where supported.

### Batching Pattern

Collect requests over a time window and send as batch:

```typescript
class BatchProcessor<T, R> {
  private queue: Request<T>[] = []
  private timer: NodeJS.Timeout

  async add(request: T): Promise<R> {
    return new Promise((resolve, reject) => {
      this.queue.push({ request, resolve, reject })

      if (this.queue.length >= MAX_BATCH_SIZE) {
        this.flush()
      } else if (!this.timer) {
        this.timer = setTimeout(() => this.flush(), BATCH_WINDOW)
      }
    })
  }

  private async flush() {
    const batch = this.queue.splice(0, MAX_BATCH_SIZE)
    const results = await this.processBatch(batch.map(b => b.request))
    batch.forEach((b, i) => b.resolve(results[i]))
  }
}
```

### Coalescing Pattern

Deduplicate identical requests within a time window:

```typescript
class RequestCoalescer<T, R> {
  private pending = new Map<string, Promise<R>>()

  async execute(key: string, fn: () => Promise<R>): Promise<R> {
    // Return existing promise if request in flight
    if (this.pending.has(key)) {
      return this.pending.get(key)!
    }

    // Create new promise and track it
    const promise = fn().finally(() => {
      this.pending.delete(key)
    })

    this.pending.set(key, promise)
    return promise
  }
}
```

## Caching Strategy Pattern

### Multi-Layer Cache Architecture

```
L1: Memory Cache (Hot)
  ↓ miss
L2: IndexedDB (Warm)
  ↓ miss
L3: Network Fetch
  ↓ success
Update all layers
```

### Cache Key Generation

```typescript
function generateCacheKey(context: string, params: any): string {
  const normalized = normalizeParams(params)
  const hash = crypto.hash(JSON.stringify(normalized))
  return `${context}:${hash}`
}

function normalizeParams(params: any): any {
  // Sort object keys
  // Lowercase addresses
  // Normalize decimals
  // Remove undefined values
}
```

### TTL Strategy by Data Type

```typescript
const TTL_STRATEGY = {
  // Static data - long cache
  'token-metadata': 86400,    // 24 hours
  'contract-abi': 604800,     // 7 days

  // Semi-dynamic data - medium cache
  'balance': 60,              // 1 minute
  'transaction': 300,         // 5 minutes

  // Dynamic data - short cache
  'gas-price': 10,            // 10 seconds
  'exchange-rate': 30,        // 30 seconds

  // User-specific - session cache
  'portfolio': 120,           // 2 minutes
  'positions': 180            // 3 minutes
}
```

### Cache Warming Pattern

```typescript
class CacheWarmer {
  async warmCache(addresses: string[]) {
    // Identify likely needed data
    const predictions = this.predictNeededData(addresses)

    // Fetch in parallel with low priority
    await Promise.allSettled(
      predictions.map(p => this.fetchWithLowPriority(p))
    )
  }

  private predictNeededData(addresses: string[]) {
    // Based on historical access patterns
    // Common token contracts
    // Frequently accessed positions
  }
}
```

## Connection Pool Management

### Purpose
Efficiently manage connections to external services with reuse and health checking.

### Pool Configuration

```typescript
interface PoolConfig {
  minConnections: 2,
  maxConnections: 10,
  idleTimeout: 30000,        // 30 seconds
  connectionTimeout: 5000,    // 5 seconds
  healthCheckInterval: 60000, // 1 minute
  acquireTimeout: 10000       // 10 seconds
}
```

### Health Check Pattern

```typescript
class ConnectionPool {
  async healthCheck(connection: Connection): Promise<boolean> {
    try {
      // Lightweight operation to verify connection
      await connection.ping()

      // Reset idle timer
      connection.lastUsed = Date.now()

      return true
    } catch (error) {
      // Mark for removal
      connection.healthy = false
      return false
    }
  }

  private async maintainPool() {
    // Remove unhealthy connections
    // Create new connections if below minimum
    // Close idle connections if above minimum
  }
}
```

## Event Publishing Pattern

### Domain Event Structure

```typescript
interface DomainEvent {
  id: string              // Unique event ID
  type: string           // Event type constant
  timestamp: number      // Unix timestamp
  source: string         // Originating context
  correlationId: string  // Request correlation
  payload: any          // Event-specific data
  metadata: {
    version: string
    userId?: string
    sessionId?: string
  }
}
```

### Event Types

```typescript
enum IntegrationEventType {
  // Connection events
  WALLET_CONNECTED = 'integration.wallet.connected',
  WALLET_DISCONNECTED = 'integration.wallet.disconnected',

  // Data events
  BALANCE_UPDATED = 'integration.balance.updated',
  TRANSACTION_DETECTED = 'integration.transaction.detected',

  // System events
  RATE_LIMIT_REACHED = 'integration.ratelimit.reached',
  CIRCUIT_OPENED = 'integration.circuit.opened',
  CACHE_INVALIDATED = 'integration.cache.invalidated'
}
```

### Publishing Pattern

```typescript
class EventPublisher {
  async publish(event: DomainEvent): Promise<void> {
    // Add to local queue first
    this.localQueue.add(event)

    // Publish to domain event bus
    await this.eventBus.publish(event)

    // Log for audit trail
    this.auditLogger.log(event)
  }

  async publishBatch(events: DomainEvent[]): Promise<void> {
    // Group by type for efficient processing
    const grouped = this.groupByType(events)

    // Publish each group
    await Promise.all(
      Object.entries(grouped).map(([type, events]) =>
        this.eventBus.publishBatch(type, events)
      )
    )
  }
}
```

## Rate Limit Management

### Token Bucket Algorithm

```typescript
class TokenBucket {
  private tokens: number
  private lastRefill: number

  constructor(
    private capacity: number,    // Max tokens
    private refillRate: number   // Tokens per second
  ) {
    this.tokens = capacity
    this.lastRefill = Date.now()
  }

  async acquire(tokens: number = 1): Promise<boolean> {
    this.refill()

    if (this.tokens >= tokens) {
      this.tokens -= tokens
      return true
    }

    // Calculate wait time
    const waitTime = (tokens - this.tokens) / this.refillRate * 1000

    if (waitTime > MAX_WAIT) {
      return false
    }

    await sleep(waitTime)
    return this.acquire(tokens)
  }

  private refill() {
    const now = Date.now()
    const elapsed = (now - this.lastRefill) / 1000
    const refillAmount = elapsed * this.refillRate

    this.tokens = Math.min(this.capacity, this.tokens + refillAmount)
    this.lastRefill = now
  }
}
```

### Coordinated Rate Limiting

```typescript
class RateLimitCoordinator {
  private limits = new Map<string, TokenBucket>()

  async executeWithLimit(
    provider: string,
    fn: () => Promise<any>
  ): Promise<any> {
    const bucket = this.limits.get(provider)

    if (!bucket) {
      return fn() // No limit configured
    }

    const acquired = await bucket.acquire()

    if (!acquired) {
      throw new RateLimitError(provider)
    }

    return fn()
  }

  // Share limit information across contexts
  publishLimitStatus(provider: string, remaining: number) {
    this.eventBus.publish({
      type: 'RATE_LIMIT_STATUS',
      provider,
      remaining,
      resetAt: this.calculateReset(provider)
    })
  }
}
```

## Error Handling Pattern

### Error Hierarchy

```typescript
class IntegrationError extends Error {
  constructor(
    message: string,
    public code: string,
    public retriable: boolean,
    public context?: any
  ) {
    super(message)
  }
}

class ConnectionError extends IntegrationError {
  retriable = true
}

class RateLimitError extends IntegrationError {
  retriable = true
  constructor(public resetAt: number) {
    super('Rate limit exceeded', 'RATE_LIMITED', true)
  }
}

class ValidationError extends IntegrationError {
  retriable = false
}

class DataError extends IntegrationError {
  retriable = false
}
```

### Error Recovery Strategies

```typescript
class ErrorRecovery {
  async handle(error: IntegrationError): Promise<any> {
    // Log error with context
    this.logger.error(error)

    // Update metrics
    this.metrics.recordError(error.code)

    // Determine recovery strategy
    if (error.retriable) {
      return this.handleRetriable(error)
    } else {
      return this.handleNonRetriable(error)
    }
  }

  private async handleRetriable(error: IntegrationError) {
    // Check circuit breaker
    if (this.circuitBreaker.shouldOpen(error)) {
      return this.fallbackStrategy.execute()
    }

    // Attempt retry with backoff
    return this.retryStrategy.execute()
  }

  private async handleNonRetriable(error: IntegrationError) {
    // Return cached data if available
    if (this.cache.has(this.context)) {
      return this.cache.get(this.context)
    }

    // Return safe default
    return this.defaultValue()
  }
}
```

## Summary

These patterns form the foundation for reliable, performant, and maintainable integrations within the Integration Domain. System Architects should:

1. **Apply patterns consistently** across all bounded contexts
2. **Adapt configurations** to specific integration requirements
3. **Monitor pattern effectiveness** through metrics
4. **Share improvements** across the domain team
5. **Document deviations** when patterns need modification

Remember: These patterns are guidelines. Always consider the specific requirements and constraints of each integration when implementing them.

---

## Related Documentation

- **[← Integration Domain README](./README.md)** - Domain overview and strategic guidance
- **[Resilience & Performance Guide](./resilience-performance.md)** - Apply these patterns with resilience strategies
- **[Testing & Security Strategy](./testing-security.md)** - Test pattern implementations thoroughly
- **[Bounded Contexts](./bounded-contexts/)** - See pattern applications in context specifications