# Asset Valuator Domain

## Domain Overview

The Asset Valuator domain is responsible for cryptocurrency and traditional asset pricing and valuation across the CygnusWealth ecosystem. It provides real-time price data, currency conversions, and standardized valuation services for all asset types.

## Core Responsibilities

### Primary Functions
1. **Real-time Price Fetching**: Retrieve current market prices from multiple sources
2. **Currency Conversion**: Convert between crypto and fiat currencies
3. **Price Caching**: Intelligent caching with configurable TTL
4. **Data Standardization**: Transform external price data to internal formats
5. **Batch Operations**: Efficient bulk price fetching for multiple assets
6. **Price Aggregation**: Combine prices from multiple sources for accuracy

### Domain Boundaries
- **Owns**: Price data retrieval, caching logic, conversion calculations, price aggregation
- **Does NOT Own**: Portfolio calculations, transaction history, wallet connections, asset balances

## Pricing Strategy

### Data Sources
- **Primary Sources**: Major price aggregators (CoinGecko, CoinMarketCap)
- **Secondary Sources**: Direct exchange APIs for specific assets
- **Fallback Sources**: Cached data when APIs unavailable
- **Custom Sources**: User-defined price feeds for unlisted assets

### Price Resolution
- Multiple source aggregation for accuracy
- Outlier detection and filtering
- Volume-weighted average pricing
- Fallback to last known good price

## Supported Asset Types

### Cryptocurrency
- Major cryptocurrencies (BTC, ETH, etc.)
- ERC-20 tokens
- SPL tokens (Solana)
- NFT floor prices
- DeFi LP tokens

### Traditional Finance
- Stocks and ETFs
- Commodities
- Forex rates
- Index funds
- Bonds and treasuries

## Caching Architecture

### Cache Layers
- **Hot Cache**: 60-second TTL for active assets
- **Warm Cache**: 5-minute TTL for less active assets
- **Cold Cache**: 1-hour TTL for stable assets
- **Historical Cache**: Persistent storage for historical data

### Cache Invalidation
- Time-based expiration
- Event-driven updates
- Manual refresh capability
- Smart pre-fetching for anticipated requests

## Data Transformation

### Standardization Rules
- Unified price format across all sources
- Consistent decimal precision
- ISO currency codes
- UTC timestamps
- Source attribution

### Currency Handling
- Support for 150+ fiat currencies
- Crypto-to-crypto conversions
- Historical exchange rates
- Real-time forex data

## Performance Optimization

### Batching Strategy
- Group API requests by source
- Minimize external API calls
- Parallel processing where possible
- Request deduplication

### Rate Limiting
- Per-source rate limit tracking
- Automatic throttling
- Queue management
- Burst handling

## Error Handling

### Common Scenarios
- API rate limits exceeded
- Network timeouts
- Invalid symbols
- Stale data detection
- Source unavailability

### Recovery Strategies
1. **Source Failover**: Automatic switch to backup sources
2. **Cache Fallback**: Use cached data when fresh data unavailable
3. **Partial Results**: Return available prices in batch requests
4. **Retry Logic**: Exponential backoff with jitter
5. **Circuit Breaker**: Temporary source disabling

## Integration Patterns

### Service Interface
The domain exposes services for price data:

#### Price Service
- Single asset price fetching
- Batch price retrieval
- Historical price data
- Price change calculations

#### Conversion Service
- Currency conversion
- Exchange rate fetching
- Multi-hop conversions
- Historical rates

#### Market Data Service
- Market capitalization
- Trading volume
- Price movements
- Market trends

## Quality Assurance

### Data Validation
- Price sanity checks
- Outlier detection
- Source reliability scoring
- Consistency verification

### Monitoring
- API health checks
- Cache hit ratios
- Response time tracking
- Error rate monitoring

## Configuration

### Configurable Parameters
- Cache TTL values
- API endpoints
- Rate limit thresholds
- Retry attempts
- Timeout values
- Price precision

### API Configuration
- API key management
- Endpoint selection
- Request priorities
- Fallback ordering

## Testing Strategy

### Test Coverage Areas
- Price fetching accuracy
- Currency conversion correctness
- Cache behavior
- Error handling
- Rate limit compliance
- Batch operation efficiency

### Testing Scenarios
- Multi-source price aggregation
- API failure handling
- Cache expiration
- High-volume batch requests
- Concurrent request handling

## Performance Targets

- Single price fetch: Under 200ms (cached), under 1s (fresh)
- Batch fetch (100 assets): Under 2 seconds
- Currency conversion: Under 100ms
- Cache hit ratio: Above 80%
- API success rate: Above 99%

## Security Considerations

- API key encryption
- Request authentication
- Data integrity verification
- Rate limit protection
- DDoS mitigation

## Future Enhancements

- Machine learning for price prediction
- Decentralized price oracles
- Custom price feed integration
- Advanced market analytics
- Real-time WebSocket feeds
- Historical data archival
- Cross-chain price discovery