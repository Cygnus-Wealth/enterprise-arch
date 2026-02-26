# Kraken Integration

> **Navigation**: [Integration Domain](../README.md) > [Bounded Contexts](../README.md#bounded-contexts) > Kraken Integration

## Bounded Context Overview

The Kraken Integration bounded context provides read-only access to Kraken centralized exchange accounts, enabling users to include their Kraken cryptocurrency holdings, margin positions, staking positions, and trade history in their aggregated portfolio. It handles HMAC-SHA512 authentication, data fetching, and transformation of Kraken-specific data into the CygnusWealth unified data models.

**For context**: This bounded context specification is part of the Integration Domain. See the [Integration Domain README](../README.md) for strategic guidance and domain-level patterns. The full architectural specification is in [Kraken CEX integration directive](../directives/kraken-cex-integration.md).

## Core Responsibilities

### Primary Functions
1. **Authentication**: HMAC-SHA512 signed requests with nonce management and optional 2FA/OTP
2. **Balance Retrieval**: Spot, margin, and staking balances via Kraken private endpoints
3. **Trade History**: Paginated trade fills and ledger entries
4. **Position Tracking**: Open margin positions and staking positions
5. **Data Transformation**: Convert Kraken responses to unified data models
6. **Asset Resolution**: Translate Kraken's non-standard asset naming (XXBT → BTC, ZUSD → USD)

### Domain Boundaries
- **Owns**: Kraken API interactions, HMAC signing, nonce lifecycle, Kraken-specific data transformation, asset pair name resolution
- **Does NOT Own**: Trading execution, order placement, deposit/withdrawal initiation, portfolio aggregation, asset pricing

## Authentication Strategy

### HMAC-SHA512 Implementation
- API key + secret provided by user (read-only permissions only)
- Signature: `HMAC-SHA512(uriPath + SHA256(nonce + postData), base64Decode(secret))`
- Monotonically increasing nonce per API key
- Web Crypto API for browser-safe HMAC computation
- Optional 2FA/OTP support

### Security Measures
- API secret never leaves the browser
- Kraken permission scoping enforces read-only at the exchange level
- Nonce prevents replay attacks
- Encrypted credential storage via IntegrationCredentials
- No passphrase required (unlike Coinbase)

## Data Fetching Capabilities

### Account Data
- Spot balances (all cryptocurrency and fiat holdings)
- Extended balances (available, held in orders, credit)
- Trade balance (margin equity, free margin, cost basis)
- Open margin positions (long/short with P&L)

### Staking Data
- Active staking positions with APY
- Staking transaction history
- Pending stake/unstake operations

### Transaction History
- Trade fills (50 per page, offset pagination)
- Ledger entries (deposits, withdrawals, trades, staking rewards)
- Closed orders

### Market Data (Public)
- Asset metadata (decimals, display names)
- Tradable asset pairs
- Ticker information (prices, volume)

## Data Transformation

### Unified Model Mapping
- Kraken balances to Balance model with Asset enrichment
- Trades and ledger entries to Transaction model with asset flow pattern (assetsIn/assetsOut/fees)
- Margin positions to Balance with position metadata
- Staking data to StakedPosition model

### Kraken-Specific Handling
- Asset name translation (XXBT → BTC, ZUSD → USD, altname resolution)
- Trading pair decomposition into base/quote assets
- Counter-based rate limit compliance
- Kraken-specific metadata namespace (kraken:holdTrade, kraken:margin, etc.)

## Integration Patterns

### Service Interface

#### Authentication Service
- Verify API key credentials via test call
- Handle 2FA/OTP challenges
- Manage credential lifecycle

#### Portfolio Service
- Assemble three virtual accounts (spot, margin, staking)
- Aggregate all balance types into unified Portfolio

#### Transaction Service
- Paginated trade history retrieval
- Ledger entry retrieval (deposits, withdrawals, rewards)

#### Staking Service
- Active staking positions
- Staking rewards history
- Pending operations

## Error Handling

### Common Error Scenarios
- Invalid API key/secret (EAPI:Invalid key)
- Invalid signature (EAPI:Invalid signature)
- Nonce errors (EAPI:Invalid nonce)
- Rate limiting (EAPI:Rate limit exceeded)
- Service throttling (EService:Throttled)
- Service unavailability (EService:Unavailable)
- Permission denied (EGeneral:Permission denied)

### Recovery Strategies
1. **Auth Failures**: Prompt re-authentication
2. **Nonce Errors**: Reset nonce counter, retry
3. **Rate Limiting**: Wait for counter decay, retry
4. **Throttling**: Wait until provided timestamp, retry
5. **Service Down**: Circuit breaker + cached data fallback
6. **Partial Failure**: Graceful degradation (return available data)

## Caching Strategy

### Cache Layers
- Asset metadata and pairs: 24-hour TTL (L1 + L2)
- Balances: 60-second TTL (L1)
- Trade history: 5-minute TTL (L1 + L2)
- Staking positions: 5-minute TTL (L1)
- Fee schedule: 1-hour TTL (L1)

### Cache Invalidation
- On credential change
- On manual refresh trigger
- On TTL expiration

## Rate Limiting

### Kraken Counter System
- Counter starts at 0, increases per call (+1 standard, +2 history)
- Decays over time based on user's verification tier
- Starter: max 15, decay -0.33/sec
- Intermediate: max 20, decay -0.5/sec
- Pro: max 20, decay -1/sec

### Optimization Strategies
- Batch asset metadata fetches
- Cache aggressively for static data
- Pace history pagination to avoid counter overflow
- Prefer single-call endpoints over paginated when possible

## Testing Strategy

### Test Coverage Areas
- HMAC signature correctness (known test vectors)
- Nonce monotonicity under rapid requests
- Asset name resolution (legacy and modern names)
- Rate limit counter behavior
- Pagination completeness
- Data transformation accuracy
- Error recovery flows
- Graceful degradation

### Integration Testing
- Full portfolio assembly from recorded fixtures
- Partial endpoint failure scenarios
- 2FA challenge flow
- Rate limit enforcement

## Configuration

### Configurable Parameters
- API base URL
- Request timeout (default 30s)
- Retry attempts (default 3)
- Rate limit tier (Starter/Intermediate/Pro)
- Cache TTLs

## Performance Targets

- Authentication verification: Under 2 seconds
- Balance fetch: Under 2 seconds
- Full portfolio assembly: Under 5 seconds
- Trade history (50 items): Under 2 seconds
- Cache hit ratio: Above 70%

## Limitations

### API Restrictions
- Read-only access (Query permissions only)
- 50 results per page for history endpoints
- Counter-based rate limiting (not RPS)
- No WebSocket for account data (HTTP polling only)

### Data Availability
- Trade history pagination for large accounts
- Staking endpoint availability may vary
- Legacy asset naming requires translation layer

## Future Enhancements

- WebSocket integration for real-time order book data
- Futures account support
- Earn/savings product tracking
- Cross-margin vs isolated margin distinction
- Sub-account support

---

## Related Documentation

- **[← Integration Domain README](../README.md)** - Domain overview and strategic guidance
- **[Kraken CEX Integration Directive](../directives/kraken-cex-integration.md)** - Full architectural specification
- **[Integration Patterns](../patterns.md)** - Apply domain patterns to this context
- **[Resilience & Performance](../resilience-performance.md)** - Implement resilience strategies
- **[Testing & Security](../testing-security.md)** - Apply testing and security standards
- **Other Bounded Contexts**: [Wallet Integration System](./wallet-integration-system.md) | [EVM Integration](./evm-integration.md) | [Solana Integration](./sol-integration.md) | [Robinhood Integration](./robinhood-integration.md)
