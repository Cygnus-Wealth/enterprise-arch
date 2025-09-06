# CygnusWealth Core Domain

## Domain Overview

The CygnusWealth Core domain serves as the main application layer and orchestration hub for the entire ecosystem. It provides the user interface, manages state, coordinates between domains, and ensures a seamless user experience for decentralized portfolio management.

## Core Responsibilities

### Primary Functions
1. **User Interface**: React-based responsive web application
2. **Domain Orchestration**: Coordinate calls between integration domains
3. **State Management**: Application-wide state with Zustand
4. **Data Aggregation**: Combine data from multiple sources into unified views
5. **User Authentication**: Local authentication and encrypted storage
6. **Portfolio Visualization**: Charts, tables, and analytics display
7. **IPFS Integration**: Decentralized backup and sync capabilities

### Domain Boundaries
- **Owns**: UI components, user interactions, orchestration logic, local storage
- **Does NOT Own**: Direct blockchain interactions, price fetching, wallet connections

## Architecture Patterns

### Component Structure
```
src/
├── components/          # Reusable UI components
│   ├── Portfolio/      # Portfolio display components
│   ├── Charts/         # Data visualization
│   ├── Wallets/        # Wallet management UI
│   └── Common/         # Shared components
├── hooks/              # Custom React hooks
├── services/           # Domain integration services
├── stores/             # Zustand state stores
├── utils/              # Helper functions
└── types/              # Local type definitions
```

### State Management
```typescript
// Zustand stores
PortfolioStore {
  - portfolios: Portfolio[]
  - activePortfolio: Portfolio | null
  - setActivePortfolio(id: string): void
  - refreshPortfolio(): Promise<void>
}

IntegrationStore {
  - connections: IntegrationConnection[]
  - addConnection(type, config): void
  - removeConnection(id): void
  - syncAll(): Promise<void>
}

UserStore {
  - preferences: UserPreferences
  - encryptedData: EncryptedStorage
  - authenticate(password): boolean
  - logout(): void
}
```

## Integration Orchestration

### Service Layer
```typescript
class PortfolioAggregator {
  async aggregatePortfolio(): Promise<Portfolio> {
    // 1. Get wallet connections from wallet-integration-system
    const wallets = await walletManager.getConnectedWallets();
    
    // 2. Fetch balances from blockchain integrations
    const evmBalances = await evmIntegration.getBalances(wallets.evm);
    const solBalances = await solIntegration.getBalances(wallets.solana);
    
    // 3. Get traditional assets from Robinhood
    const tradFi = await robinhoodIntegration.getPortfolio();
    
    // 4. Enrich with pricing data
    const prices = await assetValuator.getBatchPrices(allAssets);
    
    // 5. Transform to unified portfolio model
    return this.transformToPortfolio(data, prices);
  }
}
```

## Key Components

### Core Services
1. **PortfolioAggregator**: Combines data from all sources
2. **IntegrationManager**: Manages connections to external systems
3. **DataSyncService**: Handles IPFS backup and sync
4. **CacheManager**: Local caching for performance
5. **EncryptionService**: Client-side encryption for sensitive data

### UI Components
```typescript
// Main application components
<App>
  <AuthProvider>
    <Dashboard>
      <PortfolioOverview />
      <AssetAllocation />
      <TransactionHistory />
      <IntegrationManager />
    </Dashboard>
  </AuthProvider>
</App>
```

## External Dependencies

### Required Domains
- **data-models**: All data type definitions
- **wallet-integration-system**: Wallet connection management
- **evm-integration**: Ethereum/EVM data
- **sol-integration**: Solana data
- **robinhood-integration**: Traditional finance data
- **asset-valuator**: Price and valuation data

### Third-Party Libraries
- **React 19**: UI framework
- **Chakra UI v3**: Component library
- **Recharts**: Data visualization
- **Zustand**: State management
- **IPFS HTTP Client**: Decentralized storage

## User Workflows

### Primary User Journeys

1. **Initial Setup**
   ```
   User → Authentication → Create Portfolio → Connect Integrations
   ```

2. **Portfolio Viewing**
   ```
   User → Dashboard → Select Portfolio → View Analytics
   ```

3. **Adding Integrations**
   ```
   User → Integration Manager → Select Type → Connect → Sync Data
   ```

4. **Data Refresh**
   ```
   User → Refresh Button → Orchestrator → All Integrations → Update UI
   ```

## Data Flow

### Aggregation Pipeline
```
1. User requests portfolio refresh
2. Core queries all connected integrations
3. Each integration fetches latest data
4. Asset valuator provides current prices
5. Data transformer normalizes to data-models
6. Aggregator combines into unified portfolio
7. UI updates with new data
```

## Security & Privacy

### Client-Side Security
1. **No Server Storage**: All data stored locally in browser
2. **Encryption**: AES-256 encryption for sensitive data
3. **Password Protection**: Local authentication required
4. **No Telemetry**: Zero tracking or analytics
5. **Read-Only**: Never handles private keys

### Data Protection
```typescript
interface SecurityConfig {
  encryptionKey: string     // Derived from user password
  autoLockTimeout: number   // Default: 15 minutes
  clearOnLogout: boolean    // Default: false
  ipfsEncryption: boolean   // Default: true
}
```

## Performance Optimization

### Strategies
1. **Lazy Loading**: Load integrations on demand
2. **Memoization**: Cache computed values
3. **Virtual Scrolling**: For large transaction lists
4. **Web Workers**: Heavy computations off main thread
5. **Progressive Enhancement**: Core features work without all integrations

### Caching Strategy
```typescript
CacheConfig {
  portfolioTTL: 5 minutes
  priceTTL: 1 minute
  transactionTTL: 10 minutes
  localStorage: 10MB limit
}
```

## Error Handling

### User-Facing Errors
1. **Integration Failures**: Show partial data with warning
2. **Network Issues**: Offline mode with cached data
3. **Invalid Data**: Validation errors with helpful messages
4. **Rate Limiting**: Queue and retry with user notification

### Recovery Strategies
- Automatic retry with exponential backoff
- Fallback to cached data when available
- Graceful degradation of features
- Clear error messages with actionable steps

## Configuration

### User Preferences
```typescript
interface UserPreferences {
  theme: 'light' | 'dark' | 'system'
  currency: string
  refreshInterval: number
  notifications: NotificationSettings
  displayOptions: DisplaySettings
}
```

### Environment Configuration
```typescript
interface AppConfig {
  ipfsGateway: string
  maxRetries: number
  debugMode: boolean
  featureFlags: FeatureFlags
}
```

## Testing Strategy

### Test Coverage
1. **Unit Tests**: Components, hooks, utilities
2. **Integration Tests**: Domain service interactions
3. **E2E Tests**: User workflows with Playwright
4. **Visual Tests**: Component screenshots

### Key Test Scenarios
- Portfolio aggregation with multiple sources
- Offline mode functionality
- Error recovery flows
- Data encryption/decryption
- IPFS backup and restore

## Future Enhancements

### Planned Features
1. **Mobile App**: React Native implementation
2. **Advanced Analytics**: DeFi yield tracking, impermanent loss
3. **Tax Reporting**: Transaction categorization and reports
4. **Multi-User Support**: Family/team portfolios
5. **Automated Strategies**: Rebalancing suggestions

### Technical Improvements
- Server-side rendering for SEO
- PWA capabilities for offline use
- WebAssembly for performance-critical code
- P2P sync without IPFS dependency