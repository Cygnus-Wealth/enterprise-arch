# EVM Integration Domain

## Domain Overview

The EVM Integration domain provides read-only blockchain data access for all Ethereum Virtual Machine compatible chains. It offers React hooks and services for fetching balances, transactions, and monitoring real-time blockchain events across multiple EVM networks.

## Core Responsibilities

### Primary Functions
1. **Balance Queries**: Fetch native and ERC-20 token balances
2. **Transaction Monitoring**: Track and retrieve transaction history
3. **Real-time Updates**: WebSocket subscriptions for live data
4. **Multi-chain Support**: Handle all EVM-compatible chains
5. **Connection Management**: Smart RPC endpoint selection and fallbacks
6. **React Integration**: Custom hooks for seamless UI integration

### Domain Boundaries
- **Owns**: EVM blockchain reads, RPC management, data transformation
- **Does NOT Own**: Wallet connections, transaction signing, private keys

## Supported Chains

### Production Networks
```typescript
const SUPPORTED_CHAINS = {
  ETHEREUM: { chainId: 1, name: 'Ethereum Mainnet' },
  POLYGON: { chainId: 137, name: 'Polygon' },
  ARBITRUM: { chainId: 42161, name: 'Arbitrum One' },
  OPTIMISM: { chainId: 10, name: 'Optimism' },
  BSC: { chainId: 56, name: 'BNB Smart Chain' },
  AVALANCHE: { chainId: 43114, name: 'Avalanche C-Chain' },
  BASE: { chainId: 8453, name: 'Base' },
  FANTOM: { chainId: 250, name: 'Fantom Opera' }
}
```

## Architecture

### Core Components
```typescript
class EvmIntegration {
  // Connection management
  connect(chainId: number, rpcUrl?: string): Promise<void>
  disconnect(): void
  switchChain(chainId: number): Promise<void>
  
  // Data fetching
  getBalance(address: string): Promise<Balance>
  getTokenBalances(address: string, tokens: string[]): Promise<TokenBalance[]>
  getTransactions(address: string, options?: TxOptions): Promise<Transaction[]>
  
  // Real-time monitoring
  subscribeToBalance(address: string, callback: BalanceCallback): Unsubscribe
  subscribeToTransactions(address: string, callback: TxCallback): Unsubscribe
}
```

### React Hooks
```typescript
// Balance hooks
function useEvmBalance(address?: string, chainId?: number): {
  balance: Balance | null
  isLoading: boolean
  error: Error | null
  refetch: () => void
}

function useEvmBalanceRealTime(address?: string): {
  balance: Balance | null
  isSubscribed: boolean
  error: Error | null
}

// Transaction hooks
function useEvmTransactions(address?: string, options?: TxOptions): {
  transactions: Transaction[]
  isLoading: boolean
  error: Error | null
  hasMore: boolean
  loadMore: () => void
}

// Connection hooks
function useEvmConnect(): {
  isConnected: boolean
  chainId: number | null
  connect: (chainId: number) => Promise<void>
  disconnect: () => void
  switchChain: (chainId: number) => Promise<void>
}
```

## Data Models

### Balance Types
```typescript
interface Balance {
  address: string
  chainId: number
  native: NativeBalance
  tokens: TokenBalance[]
  lastUpdated: Date
}

interface NativeBalance {
  symbol: string  // ETH, MATIC, BNB, etc.
  balance: bigint
  decimals: number
  formatted: string
  value?: number  // USD value
}

interface TokenBalance {
  contractAddress: string
  symbol: string
  name: string
  balance: bigint
  decimals: number
  formatted: string
  standard: 'ERC20' | 'ERC721' | 'ERC1155'
  value?: number
  logo?: string
}
```

### Transaction Types
```typescript
interface EvmTransaction {
  hash: string
  chainId: number
  blockNumber: number
  timestamp: Date
  from: string
  to: string
  value: bigint
  gasUsed: bigint
  gasPrice: bigint
  status: 'success' | 'failed' | 'pending'
  type: TransactionType
  tokenTransfers?: TokenTransfer[]
  logs?: Log[]
}

interface TokenTransfer {
  contractAddress: string
  from: string
  to: string
  value: bigint
  symbol?: string
  decimals?: number
}
```

## Connection Strategy

### RPC Management
```typescript
class RpcManager {
  // Endpoint selection
  private endpoints: RpcEndpoint[] = [
    { url: 'wss://...', type: 'websocket', priority: 1 },
    { url: 'https://...', type: 'http', priority: 2 }
  ]
  
  // Connection pooling
  private connectionPool: Map<number, Connection>
  
  // Failover logic
  async getConnection(chainId: number): Promise<Connection> {
    // Try WebSocket first
    // Fall back to HTTP if WebSocket fails
    // Implement circuit breaker pattern
  }
}
```

### Performance Optimization
```typescript
interface OptimizationConfig {
  // Batching
  batchRequests: boolean
  batchSize: number
  batchDelay: number
  
  // Caching
  cacheEnabled: boolean
  cacheTTL: number
  maxCacheSize: number
  
  // Rate limiting
  requestsPerSecond: number
  concurrentRequests: number
}
```

## Real-time Features

### WebSocket Subscriptions
```typescript
class WebSocketManager {
  // Balance monitoring
  subscribeToBalance(address: string): Observable<Balance>
  
  // Block monitoring
  subscribeToBlocks(): Observable<Block>
  
  // Event monitoring
  subscribeToEvents(filter: EventFilter): Observable<Event>
  
  // Pending transaction monitoring
  subscribeToPendingTx(address: string): Observable<Transaction>
}
```

### Event Handling
```typescript
interface EventSubscription {
  event: string
  address: string
  topics: string[]
  callback: (log: Log) => void
}

// Common events
const EVENTS = {
  TRANSFER: 'Transfer(address,address,uint256)',
  APPROVAL: 'Approval(address,address,uint256)',
  SWAP: 'Swap(address,uint256,uint256,uint256,uint256,address)'
}
```

## Error Handling

### Error Types
```typescript
enum EvmErrorCode {
  RPC_ERROR = 'RPC_ERROR',
  NETWORK_ERROR = 'NETWORK_ERROR',
  INVALID_ADDRESS = 'INVALID_ADDRESS',
  CHAIN_NOT_SUPPORTED = 'CHAIN_NOT_SUPPORTED',
  RATE_LIMIT = 'RATE_LIMIT',
  INSUFFICIENT_DATA = 'INSUFFICIENT_DATA'
}

class EvmError extends Error {
  code: EvmErrorCode
  chainId?: number
  details?: any
}
```

### Recovery Strategies
1. **RPC Failures**: Automatic failover to backup endpoints
2. **Rate Limiting**: Exponential backoff with jitter
3. **Network Issues**: Offline queue with retry
4. **Invalid Data**: Validation and sanitization
5. **WebSocket Disconnect**: Automatic reconnection

## Integration Patterns

### Usage with Core
```typescript
// In cygnus-wealth-core
import { useEvmBalance, useEvmTransactions } from '@cygnus-wealth/evm-integration'

function PortfolioComponent({ address }) {
  const { balance, isLoading } = useEvmBalance(address)
  const { transactions } = useEvmTransactions(address, { limit: 10 })
  
  return (
    <div>
      <BalanceDisplay balance={balance} />
      <TransactionList transactions={transactions} />
    </div>
  )
}
```

### Data Transformation
```typescript
// Transform to data-models format
function transformToPortfolioAsset(balance: TokenBalance): Asset {
  return {
    id: `evm-${balance.contractAddress}`,
    symbol: balance.symbol,
    name: balance.name,
    type: AssetType.TOKEN,
    chain: Chain.ETHEREUM,
    balance: {
      amount: balance.balance,
      decimals: balance.decimals,
      formatted: balance.formatted
    },
    value: {
      amount: balance.value || 0,
      currency: 'USD',
      timestamp: new Date()
    }
  }
}
```

## Testing Strategy

### Unit Tests
```typescript
describe('EvmIntegration', () => {
  test('fetches balance correctly', async () => {
    const balance = await evmIntegration.getBalance(mockAddress)
    expect(balance.native.symbol).toBe('ETH')
  })
  
  test('handles RPC failure gracefully', async () => {
    // Simulate RPC failure
    // Verify fallback behavior
  })
})
```

### Integration Tests
- Multi-chain balance fetching
- WebSocket subscription lifecycle
- RPC failover scenarios
- Rate limit handling

## Configuration

### Domain Configuration
```typescript
interface EvmConfig {
  // Network settings
  defaultChainId: number
  supportedChains: number[]
  customRpcs?: Record<number, string[]>
  
  // Performance settings
  enableWebSockets: boolean
  enableBatching: boolean
  cacheTimeout: number
  
  // Limits
  maxRetries: number
  requestTimeout: number
  maxConcurrentRequests: number
}
```

### Default Configuration
```typescript
const DEFAULT_CONFIG: EvmConfig = {
  defaultChainId: 1,
  supportedChains: [1, 137, 42161, 10, 56],
  enableWebSockets: true,
  enableBatching: true,
  cacheTimeout: 60000,
  maxRetries: 3,
  requestTimeout: 30000,
  maxConcurrentRequests: 10
}
```

## Security Considerations

1. **Read-Only Access**: Never request or handle private keys
2. **RPC URL Validation**: Validate and sanitize RPC endpoints
3. **Address Validation**: Check address format before queries
4. **Rate Limiting**: Respect RPC provider limits
5. **Data Sanitization**: Clean all data from external sources

## Performance Metrics

### Target Metrics
- Balance fetch: < 500ms (cached), < 2s (fresh)
- Transaction query: < 1s for recent 100 txs
- WebSocket latency: < 100ms
- Cache hit ratio: > 70%
- RPC success rate: > 99%

## Future Enhancements

### Planned Features
1. **Layer 2 Optimization**: Special handling for L2 chains
2. **MEV Protection**: Private RPC endpoints for sensitive queries
3. **Advanced Filtering**: Complex transaction filtering
4. **Gas Optimization**: Gas price predictions and recommendations
5. **Contract Interactions**: ABI decoding and method parsing
6. **NFT Support**: Enhanced ERC-721/1155 handling