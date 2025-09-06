# Asset Valuator Domain

## Domain Overview

The Asset Valuator domain is responsible for cryptocurrency asset pricing and valuation across the CygnusWealth ecosystem. It provides real-time price data, currency conversions, and standardized valuation services.

## Core Responsibilities

### Primary Functions
1. **Real-time Price Fetching**: Retrieve current market prices from external APIs
2. **Currency Conversion**: Convert between crypto and fiat currencies
3. **Price Caching**: Intelligent caching with configurable TTL (default 60s)
4. **Data Standardization**: Transform external price data to internal formats
5. **Batch Operations**: Efficient bulk price fetching for multiple assets

### Domain Boundaries
- **Owns**: Price data retrieval, caching logic, conversion calculations
- **Does NOT Own**: Portfolio calculations, transaction history, wallet connections

## Key Components

### Core Classes
```typescript
AssetValuator {
  - getPrice(symbol: string, currency?: string): Promise<Price>
  - getBatchPrices(symbols: string[]): Promise<PriceMap>
  - convertValue(amount: number, from: string, to: string): Promise<number>
  - invalidateCache(): void
}

DataModelConverter {
  - toPriceModel(externalData: any): Price
  - toMarketData(externalData: any): MarketData
}

PriceProvider (interface) {
  - fetchPrice(symbol: string): Promise<ExternalPrice>
  - supportedAssets(): string[]
}
```

### Data Structures
```typescript
interface Price {
  symbol: string
  value: number
  currency: string
  timestamp: Date
  source: string
}

interface PriceMap {
  [symbol: string]: Price
}

interface MarketData {
  price: Price
  volume24h: number
  marketCap: number
  priceChange24h: number
}
```

## External Dependencies

### Required Services
- **CoinGecko API**: Primary price data source
- **Backup Price Providers**: Fallback sources for resilience

### Internal Dependencies
- **data-models**: Price and MarketData type definitions

## Integration Points

### Inbound Contracts
- **From cygnus-wealth-core**: 
  - `getPrice(symbol, currency)` - Get single asset price
  - `getBatchPrices(symbols)` - Get multiple asset prices
  - `convertValue(amount, from, to)` - Convert between currencies

### Outbound Contracts
- **To external APIs**:
  - HTTP REST calls to price providers
  - Rate limiting and retry logic

## Caching Strategy

```typescript
Cache Configuration {
  - Default TTL: 60 seconds
  - Max entries: 1000
  - Eviction: LRU (Least Recently Used)
  - Invalidation: Manual or TTL-based
}
```

## Error Handling

### Failure Scenarios
1. **API Unavailable**: Fallback to backup providers
2. **Rate Limiting**: Exponential backoff with retry
3. **Invalid Symbol**: Return null with appropriate error
4. **Network Timeout**: Circuit breaker pattern

### Recovery Strategies
- Cached data serves stale prices during outages
- Multiple provider fallback chain
- Graceful degradation with partial data

## Performance Considerations

### Optimization Strategies
1. **Batch Requests**: Combine multiple price requests
2. **Smart Caching**: Cache based on volatility patterns
3. **Parallel Fetching**: Concurrent API calls with limits
4. **Connection Pooling**: Reuse HTTP connections

### Metrics
- Average response time: < 200ms (cached), < 2s (fresh)
- Cache hit ratio target: > 80%
- API call reduction: 90% via caching

## Security Considerations

1. **API Key Management**: Secure storage, rotation support
2. **Rate Limiting**: Respect provider limits
3. **Data Validation**: Sanitize external data
4. **No Sensitive Data**: Prices are public information

## Future Enhancements

### Planned Features
- WebSocket support for real-time prices
- Historical price data storage
- Technical indicators calculation
- Custom price aggregation strategies
- More price provider integrations

### Scalability Considerations
- Distributed caching for multi-instance deployments
- Price data streaming architecture
- Advanced prediction models for cache warming

## Testing Strategy

### Unit Tests
- Price fetching with mocked providers
- Conversion calculations accuracy
- Cache behavior validation

### Integration Tests
- Real API interaction tests
- Fallback provider chain
- Rate limiting compliance

## Configuration

```typescript
interface AssetValuatorConfig {
  cacheTimeout: number        // Default: 60000ms
  providers: ProviderConfig[]  // Ordered by priority
  retryAttempts: number       // Default: 3
  requestTimeout: number      // Default: 5000ms
}
```

## Usage Example

```typescript
// Initialize valuator
const valuator = new AssetValuator({
  cacheTimeout: 60000,
  providers: [coinGeckoProvider, backupProvider]
});

// Get single price
const btcPrice = await valuator.getPrice('BTC', 'USD');

// Get multiple prices
const prices = await valuator.getBatchPrices(['BTC', 'ETH', 'SOL']);

// Convert value
const usdValue = await valuator.convertValue(1.5, 'BTC', 'USD');
```