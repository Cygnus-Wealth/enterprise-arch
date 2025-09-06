# Domain Contracts

## Overview

This document defines the contracts and communication patterns between domains in the CygnusWealth ecosystem. Each contract specifies the interface, data flow, and responsibilities for inter-domain communication.

## Contract Principles

1. **Dependency Direction**: Dependencies flow inward toward the core domain
2. **Data Models as Contracts**: All domains share the data-models package
3. **Interface Segregation**: Domains expose minimal, focused interfaces
4. **Async Communication**: All inter-domain calls are asynchronous
5. **Error Propagation**: Errors are transformed to domain-specific types

## Domain Dependency Graph

```
┌─────────────────────────────────────────────────────┐
│                 cygnus-wealth-core                   │
│                   (Orchestrator)                     │
└──────────┬──────────────────────────┬───────────────┘
           │                          │
           ▼                          ▼
┌──────────────────────┐    ┌────────────────────────┐
│ wallet-integration   │    │    asset-valuator      │
│      system          │    │                        │
└──────────┬───────────┘    └───────────┬────────────┘
           │                             │
           ▼                             ▼
┌──────────────────────────────────────────────────────┐
│                    data-models                        │
│                 (Shared Contracts)                    │
└──────────────────────────────────────────────────────┘
           ▲              ▲              ▲
           │              │              │
┌──────────┴─────┐ ┌─────┴──────┐ ┌────┴────────────┐
│ evm-integration│ │sol-integration│ │robinhood-      │
│                │ │              │ │integration      │
└────────────────┘ └──────────────┘ └─────────────────┘
```

## Core Domain Contracts

### Contract: Core → Wallet Integration

**Purpose**: Manage wallet connections across all chains

**Interface**:
```typescript
interface WalletIntegrationContract {
  // Commands
  connect(walletType: WalletType, chain: Chain): Promise<WalletConnection>
  disconnect(walletId: string): Promise<void>
  switchChain(walletId: string, chainId: number): Promise<void>
  
  // Queries
  getConnectedWallets(): WalletConnection[]
  getWalletsByChain(chain: Chain): WalletConnection[]
  
  // Events
  onWalletConnected: EventEmitter<WalletConnection>
  onWalletDisconnected: EventEmitter<string>
  onChainChanged: EventEmitter<{walletId: string, chainId: number}>
}
```

**Data Flow**:
```
Core → Request Connection → Wallet System
Core ← Return Connection ← Wallet System
Core ← Emit Events ← Wallet System
```

### Contract: Core → Asset Valuator

**Purpose**: Get pricing and valuation data

**Interface**:
```typescript
interface AssetValuatorContract {
  // Queries
  getPrice(symbol: string, currency?: string): Promise<Price>
  getBatchPrices(symbols: string[]): Promise<Map<string, Price>>
  convertValue(amount: number, from: string, to: string): Promise<number>
  getMarketData(symbol: string): Promise<MarketData>
  
  // Cache Management
  invalidateCache(symbols?: string[]): void
  getCacheStatus(): CacheStatus
}
```

**Data Flow**:
```
Core → Request Prices → Asset Valuator
Asset Valuator → Fetch from API → External Service
Asset Valuator → Cache Result → Internal Cache
Core ← Return Prices ← Asset Valuator
```

### Contract: Core → EVM Integration

**Purpose**: Fetch EVM blockchain data

**Interface**:
```typescript
interface EvmIntegrationContract {
  // Queries
  getBalance(address: string, chainId?: number): Promise<Balance>
  getTokenBalances(address: string, tokens: string[]): Promise<TokenBalance[]>
  getTransactions(address: string, options?: TxOptions): Promise<Transaction[]>
  
  // Real-time
  subscribeToBalance(address: string, callback: BalanceCallback): Unsubscribe
  subscribeToTransactions(address: string, callback: TxCallback): Unsubscribe
  
  // Chain Management
  setActiveChain(chainId: number): void
  getSupportedChains(): ChainConfig[]
}
```

**Data Flow**:
```
Core → Get Wallet Address → Wallet System
Core → Request Balance(address) → EVM Integration
EVM → Query Blockchain → RPC Provider
EVM → Transform Data → Data Models Format
Core ← Return Balance ← EVM Integration
```

### Contract: Core → Solana Integration

**Purpose**: Fetch Solana blockchain data

**Interface**:
```typescript
interface SolanaIntegrationContract {
  // Queries
  getPortfolioSnapshot(address: string): Promise<PortfolioSnapshot>
  getTokenBalances(address: string): Promise<TokenBalance[]>
  getNFTCollection(address: string): Promise<NFT[]>
  getTransactionHistory(address: string, limit?: number): Promise<Transaction[]>
  
  // Real-time
  subscribeToBalance(address: string): Observable<BalanceUpdate>
  subscribeToTransactions(address: string): Observable<Transaction>
  
  // Performance
  getCacheMetrics(): CacheMetrics
  getCircuitBreakerStatus(): CircuitBreakerStatus
}
```

**Data Flow**:
```
Core → Get Wallet PublicKey → Wallet System
Core → Request Portfolio → Solana Integration
Solana → Check Cache → LRU Cache
Solana → Fetch Data → RPC Client Pool
Solana → Apply Domain Logic → Domain Aggregates
Core ← Return Portfolio ← Solana Integration
```

### Contract: Core → Robinhood Integration

**Purpose**: Fetch traditional finance data

**Interface**:
```typescript
interface RobinhoodIntegrationContract {
  // Authentication
  authenticate(credentials: Credentials): Promise<AuthResult>
  handleMFA(code: string): Promise<AuthResult>
  refreshSession(): Promise<void>
  logout(): Promise<void>
  
  // Queries
  getPortfolio(): Promise<Portfolio>
  getPositions(): Promise<Position[]>
  getQuotes(symbols: string[]): Promise<Quote[]>
  getOrders(options?: OrderOptions): Promise<Order[]>
  
  // State
  isAuthenticated(): boolean
  getAccountInfo(): Promise<AccountInfo>
}
```

**Data Flow**:
```
Core → Initiate Auth → Robinhood Integration
Robinhood → OAuth Flow → Robinhood API
Core ← Auth Result ← Robinhood Integration
Core → Request Portfolio → Robinhood Integration
Robinhood → Fetch Data → Robinhood API
Robinhood → Transform → Data Models Format
Core ← Portfolio Data ← Robinhood Integration
```

## Integration Domain Contracts

### Contract: EVM/Solana → Data Models

**Purpose**: Use standardized data structures

**Interface**:
```typescript
// All domains import and use these types
import {
  Asset,
  Balance,
  Transaction,
  Portfolio,
  Chain,
  AssetType,
  IntegrationSource
} from '@cygnus-wealth/data-models'
```

**Usage Pattern**:
```typescript
// In EVM Integration
function transformToAsset(tokenBalance: RawTokenData): Asset {
  return {
    id: generateId(tokenBalance),
    symbol: tokenBalance.symbol,
    type: AssetType.TOKEN,
    chain: Chain.ETHEREUM,
    balance: transformBalance(tokenBalance),
    // ... map to data-models structure
  }
}
```

### Contract: EVM/Solana → Wallet System

**Purpose**: Get connected wallet addresses

**Interface**:
```typescript
interface WalletQueryContract {
  // Queries only - no modifications
  getWalletsByChain(chain: Chain): WalletConnection[]
  getActiveAddress(chain: Chain): string | null
  isWalletConnected(chain: Chain): boolean
}
```

**Usage Pattern**:
```typescript
// In EVM Integration
async function fetchUserBalance(): Promise<Balance> {
  const address = walletSystem.getActiveAddress(Chain.ETHEREUM)
  if (!address) throw new NoWalletError()
  
  return this.getBalance(address)
}
```

### Contract: All Domains → Asset Valuator

**Purpose**: Enrich data with pricing information

**Interface**:
```typescript
interface PricingContract {
  getPrice(symbol: string): Promise<Price>
  getBatchPrices(symbols: string[]): Promise<Map<string, Price>>
}
```

**Usage Pattern**:
```typescript
// In any integration domain
async function enrichWithPrices(assets: Asset[]): Promise<Asset[]> {
  const symbols = assets.map(a => a.symbol)
  const prices = await assetValuator.getBatchPrices(symbols)
  
  return assets.map(asset => ({
    ...asset,
    value: {
      amount: asset.balance.amount * prices.get(asset.symbol).value,
      currency: 'USD',
      timestamp: new Date()
    }
  }))
}
```

## Event Contracts

### Event Bus Contract

**Purpose**: Loosely coupled event communication

**Events**:
```typescript
interface DomainEvents {
  // Wallet Events
  'wallet:connected': { wallet: WalletConnection }
  'wallet:disconnected': { walletId: string }
  'wallet:accountChanged': { walletId: string, accounts: string[] }
  'wallet:chainChanged': { walletId: string, chainId: number }
  
  // Portfolio Events
  'portfolio:updated': { portfolioId: string, changes: Change[] }
  'portfolio:syncing': { portfolioId: string, source: IntegrationSource }
  'portfolio:syncComplete': { portfolioId: string, success: boolean }
  
  // Price Events
  'price:updated': { symbol: string, price: Price }
  'price:batchUpdated': { prices: Map<string, Price> }
  
  // Integration Events
  'integration:connected': { type: IntegrationType, id: string }
  'integration:disconnected': { type: IntegrationType, id: string }
  'integration:error': { type: IntegrationType, error: Error }
}
```

**Usage**:
```typescript
// Publishing events
eventBus.emit('wallet:connected', { wallet: newConnection })

// Subscribing to events
eventBus.on('wallet:connected', ({ wallet }) => {
  // React to wallet connection
  this.syncPortfolio(wallet.chain)
})
```

## Error Contracts

### Error Transformation

**Purpose**: Convert domain errors to common format

**Pattern**:
```typescript
interface ErrorContract {
  code: string
  message: string
  domain: string
  originalError?: any
  recoverable: boolean
  userAction?: string
}

// Each domain transforms its errors
function transformError(domainError: any): ErrorContract {
  return {
    code: mapErrorCode(domainError),
    message: getUserMessage(domainError),
    domain: 'evm-integration',
    originalError: domainError,
    recoverable: isRecoverable(domainError),
    userAction: getSuggestedAction(domainError)
  }
}
```

## Performance Contracts

### SLA Definitions

**Response Time Contracts**:
```typescript
interface PerformanceSLA {
  // Maximum response times (ms)
  walletConnection: 5000      // 5 seconds
  balanceFetch: 2000          // 2 seconds (fresh)
  balanceCached: 100          // 100ms (cached)
  priceFetch: 1000           // 1 second
  portfolioAggregate: 3000   // 3 seconds
  transactionQuery: 2000     // 2 seconds
}
```

### Caching Contracts

**Cache Coordination**:
```typescript
interface CacheContract {
  // Cache keys follow pattern: domain:type:identifier
  getCacheKey(domain: string, type: string, id: string): string
  
  // Coordinated cache invalidation
  invalidate(pattern: string): void
  
  // Cache TTL agreements
  getTTL(dataType: string): number
}

const CACHE_TTL = {
  'price': 60_000,           // 1 minute
  'balance': 300_000,        // 5 minutes
  'transaction': 600_000,    // 10 minutes
  'portfolio': 300_000,      // 5 minutes
  'nft': 3600_000           // 1 hour
}
```

## Testing Contracts

### Integration Test Contracts

**Mock Providers**:
```typescript
interface MockProviderContract {
  // Each domain provides mock implementations
  createMockProvider(): any
  createMockData(): any
  simulateError(type: string): void
  reset(): void
}
```

**Test Data Contracts**:
```typescript
interface TestDataContract {
  // Shared test data following data-models
  getMockAsset(): Asset
  getMockPortfolio(): Portfolio
  getMockTransaction(): Transaction
  getMockWallet(): WalletConnection
}
```

## Version Compatibility

### Semantic Versioning Contract

**Rules**:
1. **Major**: Breaking changes to contracts
2. **Minor**: New optional contract methods
3. **Patch**: Implementation changes only

**Compatibility Matrix**:
```typescript
const COMPATIBILITY = {
  'data-models': {
    '1.x': ['core@1.x', 'evm@1.x', 'sol@1.x'],
    '2.x': ['core@2.x', 'evm@2.x', 'sol@2.x']
  },
  'core': {
    '1.x': ['wallet@1.x', 'valuator@1.x'],
    '2.x': ['wallet@2.x', 'valuator@2.x']
  }
}
```

## Contract Evolution

### Adding New Contracts

**Process**:
1. Define interface in data-models
2. Implement in provider domain
3. Add optional support in consumer
4. Make required in next major version

### Deprecating Contracts

**Process**:
1. Mark as deprecated with warning
2. Provide migration guide
3. Support both old and new for one major version
4. Remove in next major version

```typescript
interface DeprecationNotice {
  method: string
  deprecatedIn: string
  removeIn: string
  alternative: string
  migrationGuide: string
}
```