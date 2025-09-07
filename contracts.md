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
│                cygnus-wealth-app                     │
│               (Experience Domain)                    │
└────────┬──────────────────┬──────────────────────────┘
         │                  │
         │                  ▼
         │    ┌──────────────────────────┐
         │    │ wallet-integration-system│
         │    │   (Integration Domain)    │
         │    └──────────┬────────────────┘
         │               │ provides addresses
         ▼               ▼
┌─────────────────────────────────────────────────────┐
│            portfolio-aggregation                     │
│              (Portfolio Domain)                      │
└──────┬─────────────────┬──────────────────┬─────────┘
       │                 │                  │
       ▼                 ▼                  ▼
┌──────────────┐  ┌──────────────┐  ┌─────────────────────────┐
│asset-valuator│  │evm-integration│  │sol-integration/robinhood│
│(Portfolio)   │  │(Integration)  │  │     (Integration)       │
└──────┬───────┘  └──────┬────────┘  └──────────┬──────────────┘
       │                 │                       │
       ▼                 ▼                       ▼
┌──────────────────────────────────────────────────────────────┐
│                        data-models                           │
│                     (Contract Domain)                        │
└──────────────────────────────────────────────────────────────┘
```

## Primary Contracts

### Contract: App → Wallet Integration

**Purpose**: Manage wallet connections from the UI

**Interface**:
```
WalletConnectionContract {
  // Commands
  connectWallet(type, options): Connection
  disconnectWallet(id): void
  switchChain(walletId, chainId): void
  
  // Queries
  getAvailableWallets(): WalletList
  getConnectedWallets(): ConnectionList
  
  // Events
  onWalletConnected: Event
  onWalletDisconnected: Event
  onChainChanged: Event
}
```

**Data Flow**:
1. User initiates wallet connection in app
2. App calls wallet-integration-system to connect
3. Wallet-integration handles provider interaction
4. Returns connection status to app
5. Addresses available for portfolio aggregation

### Contract: Portfolio Aggregation → Wallet Integration

**Purpose**: Get connected wallet addresses for data fetching

**Interface**:
```
WalletIntegrationContract {
  // Queries
  getConnectedAddresses(): AddressList
  getAddressesByChain(chain): AddressList
  getWalletMetadata(address): WalletInfo
  
  // Events
  onAddressConnected: Event
  onAddressDisconnected: Event
}
```

**Data Flow**:
1. Portfolio aggregation requests connected addresses
2. Wallet integration returns list of addresses with chain info
3. Portfolio aggregation uses addresses to coordinate blockchain queries

### Contract: Portfolio Aggregation → EVM/Sol Integration

**Purpose**: Fetch blockchain data for given addresses

**Interface**:
```
BlockchainIntegrationContract {
  // Queries
  getBalances(addresses): BalanceList
  getTransactions(addresses, options): TransactionList
  getTokens(addresses): TokenList
  getNFTs(addresses): NFTList
  
  // Real-time
  subscribeToUpdates(addresses): Subscription
}
```

**Data Flow**:
1. Portfolio aggregation provides addresses from wallet integration
2. Blockchain integrations fetch on-chain data
3. Data transformed to unified models
4. Returned to portfolio aggregation for combining

### Contract: Portfolio Aggregation → Asset Valuator

**Purpose**: Get pricing and valuation data

**Interface**:
```
AssetValuatorContract {
  // Queries
  getPrice(symbol, currency): Price
  getBatchPrices(symbols): PriceMap
  convertValue(amount, from, to): ConvertedValue
  getMarketData(symbol): MarketData
  
  // Cache Management
  invalidateCache(symbols): void
  getCacheStatus(): CacheStatus
}
```

**Data Flow**:
1. Portfolio aggregation sends assets for pricing
2. Asset valuator fetches/caches prices
3. Returns enriched valuations
4. Portfolio aggregation combines with quantities

### Contract: App → Portfolio Aggregation

**Purpose**: Orchestrate complete portfolio fetching

**Interface**:
```
PortfolioAggregationContract {
  // Commands
  refreshPortfolio(): Portfolio
  addIntegration(type, config): void
  removeIntegration(id): void
  
  // Queries
  getPortfolio(): Portfolio
  getAssets(): AssetList
  getTransactions(filter): TransactionList
  
  // Events
  onPortfolioUpdated: Event
  onSyncComplete: Event
  onError: Event
}
```

**Data Flow**:
1. App requests portfolio refresh
2. Portfolio aggregation orchestrates all integrations
3. Combines and deduplicates data
4. Returns unified portfolio to app

## Integration Patterns

### Command Pattern
All state-changing operations use commands:
- Connect wallet
- Refresh portfolio
- Add integration
- Clear cache

### Query Pattern
All data fetching uses queries:
- Get balances
- Get prices
- Get transactions
- Get portfolio

### Event Pattern
Async notifications use events:
- Wallet connected/disconnected
- Portfolio updated
- Sync completed
- Errors occurred

## Error Handling Contracts

### Error Types
Each domain defines its own error types:
```
DomainError {
  code: ErrorCode
  message: string
  domain: DomainName
  originalError?: any
  timestamp: Date
}
```

### Error Propagation
1. Errors caught at domain boundary
2. Transformed to domain-specific type
3. Logged with context
4. Propagated to caller
5. UI shows user-friendly message

## Data Transformation Contracts

### Integration → Data Models
All integrations must transform to unified models:
- External API response → Unified Asset
- Blockchain data → Unified Transaction
- Price data → Unified Price

### Validation Rules
Data must be validated at boundaries:
- Required fields present
- Correct data types
- Valid ranges
- Consistent units

## Performance Contracts

### Response Time SLAs
- Wallet connection: < 3 seconds
- Balance fetch: < 2 seconds per chain
- Price fetch: < 500ms cached, < 2s fresh
- Full portfolio: < 5 seconds

### Caching Contracts
Each domain specifies cache behavior:
- TTL for different data types
- Invalidation triggers
- Cache size limits
- Fallback strategies

## Security Contracts

### Read-Only Guarantee
Integration domains guarantee:
- Never request private keys
- Never sign transactions
- Never modify external state
- Only read public data

### Data Privacy
- No external telemetry
- Local-only storage
- Encrypted sensitive data
- No third-party sharing

## Versioning Contracts

### Semantic Versioning
All contracts follow semver:
- Major: Breaking changes
- Minor: New features
- Patch: Bug fixes

### Backward Compatibility
- Deprecation warnings
- Migration guides
- Compatibility layers
- Graceful degradation

## Testing Contracts

### Contract Tests
Each domain provides:
- Contract test suites
- Mock implementations
- Test fixtures
- Integration tests

### Test Coverage Requirements
- Unit tests: 80%+
- Integration tests: Critical paths
- Contract tests: All interfaces
- E2E tests: User journeys

## Future Contract Extensions

### Planned Additions
- WebSocket subscriptions
- Batch operations
- Streaming responses
- GraphQL interfaces

### Extension Points
Contracts designed for extension:
- New integration types
- Additional data fields
- New event types
- Performance optimizations