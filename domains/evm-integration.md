# EVM Integration Domain

## Domain Overview

The EVM Integration domain provides read-only blockchain data access for all Ethereum Virtual Machine compatible chains. It handles wallet tracking, balance queries, and transaction monitoring across multiple EVM networks, transforming blockchain data into the unified data model format.

## Core Responsibilities

### Primary Functions
1. **Wallet Tracking**: Monitor specified addresses across EVM chains
2. **Balance Queries**: Fetch native and ERC-20 token balances
3. **Transaction Monitoring**: Track and retrieve transaction history
4. **Real-time Updates**: WebSocket subscriptions for live data
5. **Multi-chain Support**: Handle all EVM-compatible chains
6. **Connection Management**: Smart RPC endpoint selection and fallbacks
7. **Data Transformation**: Convert blockchain data to unified models

### Domain Boundaries
- **Owns**: EVM blockchain reads, RPC management, address tracking, data transformation
- **Does NOT Own**: Portfolio aggregation, transaction signing, private keys, UI components

## Supported Chains

### Production Networks
- Ethereum Mainnet
- Polygon
- Arbitrum One
- Optimism
- BNB Smart Chain
- Avalanche C-Chain
- Base
- Fantom Opera

## Architecture

### Core Components

#### Connection Management
- Connect to specific chains with configurable RPC endpoints
- Handle disconnections and chain switching
- Manage connection pooling and failover

#### Data Fetching
- Query native and token balances
- Retrieve transaction history with pagination
- Batch requests for efficiency

#### Real-time Monitoring
- WebSocket subscriptions for balance updates
- Monitor incoming transactions
- Track pending transaction status

### Service Interface

The domain exposes services for data fetching, not UI components:

#### Balance Service
- Fetch balances for tracked addresses
- Support batch queries for efficiency
- Cache results with configurable TTL

#### Transaction Service
- Retrieve transaction history
- Support pagination and filtering
- Monitor pending transactions

#### Tracking Service
- Add/remove addresses to monitor
- Configure per-address settings
- Track across multiple chains simultaneously

## Data Models

### Balance Information
- Native token balances (ETH, MATIC, BNB, etc.)
- ERC-20 token holdings
- NFT collections (ERC-721, ERC-1155)
- Last updated timestamps for cache management

### Transaction Data
- Transaction hash and status
- Block number and timestamp
- From/to addresses
- Value transferred
- Gas usage and costs
- Token transfer events
- Contract interaction logs

## Connection Strategy

### RPC Management
- Priority-based endpoint selection (WebSocket preferred over HTTP)
- Connection pooling for efficiency
- Automatic failover to backup endpoints
- Circuit breaker pattern for failing endpoints

### Performance Optimization
- Request batching to reduce RPC calls
- Intelligent caching with configurable TTL
- Rate limiting to respect provider limits
- Concurrent request management

## Real-time Features

### WebSocket Subscriptions
- Balance monitoring for tracked addresses
- New block notifications
- Smart contract event monitoring
- Pending transaction tracking

### Event Handling
- Monitor common events (transfers, approvals, swaps)
- Filter events by address and topics
- Handle subscription lifecycle management
- Automatic reconnection on disconnect

## Error Handling

### Common Error Scenarios
- RPC connection failures
- Network timeouts
- Invalid addresses
- Unsupported chains
- Rate limit exceeded
- Insufficient or corrupted data

### Recovery Strategies
1. **RPC Failures**: Automatic failover to backup endpoints
2. **Rate Limiting**: Exponential backoff with jitter
3. **Network Issues**: Offline queue with retry
4. **Invalid Data**: Validation and sanitization
5. **WebSocket Disconnect**: Automatic reconnection

## Integration Patterns

### Usage with Portfolio Aggregation

The portfolio-aggregation domain calls this integration to fetch EVM data:

1. **Address List**: Aggregation provides addresses to track
2. **Data Fetch**: Integration queries blockchain for each address
3. **Transformation**: Convert to unified data-models format
4. **Return**: Provide normalized data to aggregation service

### Data Transformation

All blockchain data is transformed to the unified data-models format:

- **Native tokens**: ETH, MATIC, BNB → unified Asset model
- **ERC-20 tokens**: Standard token data → unified Asset model
- **NFTs**: ERC-721/1155 → unified NFT model
- **Transactions**: Chain-specific → unified Transaction model

This ensures consistent data structure regardless of the source chain.

## Testing Strategy

### Test Coverage Areas
- Balance fetching accuracy
- Transaction history retrieval
- Multi-chain support
- WebSocket subscription lifecycle
- RPC failover scenarios
- Rate limit handling
- Error recovery mechanisms
- Data transformation correctness

## Configuration

### Configurable Parameters
- Default chain selection
- Supported chain list
- Custom RPC endpoints
- WebSocket enablement
- Request batching settings
- Cache timeout values
- Retry limits and timeouts
- Concurrent request limits

## Security Considerations

1. **Read-Only Access**: Never request or handle private keys
2. **RPC URL Validation**: Validate and sanitize RPC endpoints
3. **Address Validation**: Check address format before queries
4. **Rate Limiting**: Respect RPC provider limits
5. **Data Sanitization**: Clean all data from external sources

## Performance Targets

- Balance fetch: Sub-500ms for cached data, under 2 seconds for fresh queries
- Transaction retrieval: Under 1 second for recent transactions
- WebSocket latency: Sub-100ms for real-time updates
- Cache effectiveness: 70%+ hit ratio
- Service reliability: 99%+ success rate

## Future Enhancements

### Planned Features
1. **Layer 2 Optimization**: Special handling for L2 chains
2. **MEV Protection**: Private RPC endpoints for sensitive queries
3. **Advanced Filtering**: Complex transaction filtering
4. **Gas Optimization**: Gas price predictions and recommendations
5. **Contract Interactions**: ABI decoding and method parsing
6. **NFT Support**: Enhanced ERC-721/1155 handling