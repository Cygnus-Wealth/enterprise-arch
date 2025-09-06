# Data Models Domain

## Domain Overview

The Data Models domain serves as the foundational contract layer for the entire CygnusWealth ecosystem. It defines the ubiquitous language through TypeScript interfaces and types that ensure consistency, type safety, and clear communication between all domains.

## Core Responsibilities

### Primary Functions
1. **Type Definitions**: Define all shared data structures
2. **Domain Contracts**: Establish interfaces between domains
3. **Data Validation**: Provide validation schemas and rules
4. **Standardization**: Ensure consistent data representation
5. **Versioning**: Manage data model evolution

### Domain Boundaries
- **Owns**: All shared type definitions, interfaces, enums, constants
- **Does NOT Own**: Business logic, data fetching, UI components, persistence

## Core Data Models

### Portfolio Models
```typescript
interface Portfolio {
  id: string
  name: string
  owner: string
  assets: Asset[]
  totalValue: Value
  lastUpdated: Date
  sources: IntegrationSource[]
}

interface Asset {
  id: string
  symbol: string
  name: string
  type: AssetType
  chain?: Chain
  balance: Balance
  value: Value
  metadata: AssetMetadata
}

interface Balance {
  amount: bigint
  decimals: number
  formatted: string
}

interface Value {
  amount: number
  currency: string
  timestamp: Date
}
```

### Transaction Models
```typescript
interface Transaction {
  id: string
  hash?: string
  type: TransactionType
  status: TransactionStatus
  from: Address
  to: Address
  asset: Asset
  amount: Balance
  fee?: Balance
  timestamp: Date
  source: IntegrationSource
  metadata: TransactionMetadata
}

enum TransactionType {
  TRANSFER = 'transfer',
  SWAP = 'swap',
  DEPOSIT = 'deposit',
  WITHDRAWAL = 'withdrawal',
  MINT = 'mint',
  BURN = 'burn',
  TRADE = 'trade'
}

enum TransactionStatus {
  PENDING = 'pending',
  CONFIRMED = 'confirmed',
  FAILED = 'failed',
  CANCELLED = 'cancelled'
}
```

### Market Data Models
```typescript
interface MarketData {
  symbol: string
  price: Price
  volume24h: number
  marketCap: number
  priceChange24h: number
  high24h: number
  low24h: number
  circulatingSupply: number
  totalSupply: number
  lastUpdated: Date
}

interface Price {
  value: number
  currency: string
  source: string
  confidence: number
  timestamp: Date
}

interface PriceHistory {
  symbol: string
  interval: TimeInterval
  data: PricePoint[]
}

interface PricePoint {
  timestamp: Date
  open: number
  high: number
  low: number
  close: number
  volume: number
}
```

### Chain & Network Models
```typescript
enum Chain {
  ETHEREUM = 'ethereum',
  POLYGON = 'polygon',
  ARBITRUM = 'arbitrum',
  OPTIMISM = 'optimism',
  BSC = 'bsc',
  AVALANCHE = 'avalanche',
  SOLANA = 'solana',
  SUI = 'sui',
  BITCOIN = 'bitcoin'
}

interface ChainConfig {
  id: Chain
  name: string
  symbol: string
  rpcUrls: string[]
  explorerUrl: string
  nativeCurrency: Currency
  chainId?: number  // For EVM chains
}

interface Network {
  chain: Chain
  isTestnet: boolean
  isConnected: boolean
  blockHeight: number
  gasPrice?: bigint
}
```

### Integration Models
```typescript
interface IntegrationSource {
  id: string
  type: IntegrationType
  name: string
  status: IntegrationStatus
  connectedAt: Date
  lastSync: Date
  config: IntegrationConfig
}

enum IntegrationType {
  WALLET = 'wallet',
  CEX = 'cex',
  DEX = 'dex',
  DEFI = 'defi',
  MANUAL = 'manual'
}

enum IntegrationStatus {
  CONNECTED = 'connected',
  DISCONNECTED = 'disconnected',
  ERROR = 'error',
  SYNCING = 'syncing'
}

interface IntegrationConfig {
  [key: string]: any  // Domain-specific configuration
}
```

### Wallet Models
```typescript
interface Wallet {
  address: Address
  chain: Chain
  type: WalletType
  label?: string
  isConnected: boolean
  provider?: string
}

interface Address {
  value: string
  chain: Chain
  format: AddressFormat
}

enum WalletType {
  SOFTWARE = 'software',
  HARDWARE = 'hardware',
  MULTI_SIG = 'multi_sig',
  SMART_CONTRACT = 'smart_contract'
}

enum AddressFormat {
  EVM = 'evm',
  SOLANA = 'solana',
  BITCOIN = 'bitcoin',
  SUI = 'sui'
}
```

## Utility Types

### Common Types
```typescript
type UUID = string
type HexString = string
type Base64String = string
type ISODateString = string
type CurrencyCode = string

interface Pagination {
  page: number
  pageSize: number
  total: number
  hasNext: boolean
  hasPrevious: boolean
}

interface ApiResponse<T> {
  data: T
  error?: Error
  metadata: ResponseMetadata
}

interface Error {
  code: string
  message: string
  details?: any
}
```

### Validation Types
```typescript
interface ValidationRule<T> {
  validate: (value: T) => boolean
  message: string
}

interface ValidationSchema<T> {
  rules: ValidationRule<T>[]
  required: boolean
}
```

## Constants & Enums

### Asset Classifications
```typescript
enum AssetType {
  CRYPTOCURRENCY = 'cryptocurrency',
  TOKEN = 'token',
  NFT = 'nft',
  STOCK = 'stock',
  ETF = 'etf',
  COMMODITY = 'commodity',
  FIAT = 'fiat'
}

enum TokenStandard {
  ERC20 = 'erc20',
  ERC721 = 'erc721',
  ERC1155 = 'erc1155',
  SPL = 'spl',
  BEP20 = 'bep20'
}
```

### Time Constants
```typescript
enum TimeInterval {
  MINUTE_1 = '1m',
  MINUTE_5 = '5m',
  MINUTE_15 = '15m',
  HOUR_1 = '1h',
  HOUR_4 = '4h',
  DAY_1 = '1d',
  WEEK_1 = '1w',
  MONTH_1 = '1M'
}

const CACHE_DURATIONS = {
  PRICE: 60_000,        // 1 minute
  BALANCE: 300_000,     // 5 minutes
  TRANSACTION: 600_000, // 10 minutes
  METADATA: 3600_000   // 1 hour
} as const
```

## Version Management

### Versioning Strategy
```typescript
interface ModelVersion {
  version: string  // semver format
  deprecated?: boolean
  migrationPath?: string
}

// Version compatibility matrix
const COMPATIBILITY = {
  '1.0.0': ['1.0.x'],
  '1.1.0': ['1.0.x', '1.1.x'],
  '2.0.0': ['2.0.x']  // Breaking change
}
```

## Type Guards & Validators

### Type Guards
```typescript
function isAsset(obj: any): obj is Asset {
  return obj && 
    typeof obj.id === 'string' &&
    typeof obj.symbol === 'string' &&
    obj.balance !== undefined
}

function isEvmAddress(address: string): boolean {
  return /^0x[a-fA-F0-9]{40}$/.test(address)
}

function isSolanaAddress(address: string): boolean {
  return /^[1-9A-HJ-NP-Za-km-z]{32,44}$/.test(address)
}
```

## Migration Patterns

### Schema Evolution
```typescript
// Migration example from v1 to v2
function migratePortfolioV1ToV2(v1: PortfolioV1): PortfolioV2 {
  return {
    ...v1,
    version: '2.0.0',
    newField: getDefaultValue(),
    // Transform old field
    assets: v1.holdings.map(transformAsset)
  }
}
```

## Usage Guidelines

### Best Practices
1. **Immutability**: Treat all data models as immutable
2. **Validation**: Always validate external data against schemas
3. **Type Safety**: Leverage TypeScript's type system fully
4. **Documentation**: Document all model changes and migrations
5. **Backward Compatibility**: Maintain compatibility when possible

### Anti-Patterns to Avoid
- Adding business logic to data models
- Circular dependencies between types
- Using `any` type without justification
- Breaking changes without version bump
- Inconsistent naming conventions

## Testing Strategies

### Type Testing
```typescript
// Type-level tests using TypeScript
type _TestAssetHasBalance = Asset['balance'] // Should not error
type _TestChainEnum = Chain.ETHEREUM // Should resolve to 'ethereum'
```

### Runtime Validation Tests
```typescript
describe('Asset Validation', () => {
  test('validates valid asset', () => {
    const asset = createMockAsset()
    expect(isAsset(asset)).toBe(true)
  })
})
```

## Documentation Standards

### Type Documentation
```typescript
/**
 * Represents a blockchain asset held in a portfolio
 * @property id - Unique identifier for the asset
 * @property symbol - Trading symbol (e.g., 'BTC', 'ETH')
 * @property chain - Optional blockchain where asset resides
 */
interface Asset {
  // ...
}
```

## Export Strategy

### Package Exports
```typescript
// index.ts
export * from './models/portfolio'
export * from './models/transaction'
export * from './models/market'
export * from './models/chain'
export * from './models/integration'
export * from './models/wallet'
export * from './constants'
export * from './validators'
export * from './types'
```