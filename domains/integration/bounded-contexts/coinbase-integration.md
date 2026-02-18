# Coinbase Integration

> **Navigation**: [Integration Domain](../README.md) > [Bounded Contexts](../README.md#bounded-contexts) > Coinbase Integration

## Bounded Context Overview

The Coinbase Integration bounded context provides read-only access to Coinbase accounts via the official Coinbase Advanced Trade API and Coinbase Platform API. It enables users to include their centralized exchange holdings — spot balances, staking positions, earn rewards, and transaction history — in their aggregated CygnusWealth portfolio. It handles authentication, data fetching, and transformation of all Coinbase asset types to the unified data model.

**For context**: This bounded context specification is part of the Integration Domain. See the [Integration Domain README](../README.md) for strategic guidance and domain-level patterns.

## Core Responsibilities

### Primary Functions
1. **Authentication**: OAuth2 with PKCE for client-side safe authorization
2. **Account Retrieval**: Fetch all Coinbase accounts and balances
3. **Position Tracking**: Spot holdings, staking positions, Coinbase Earn rewards
4. **Transaction History**: Trades, sends, receives, conversions, staking rewards
5. **Market Data**: Real-time prices and trading pair information
6. **Data Transformation**: Convert to standardized CygnusWealth data models

### Domain Boundaries
- **Owns**: Coinbase API interactions, OAuth tokens, data transformation, rate limit management
- **Does NOT Own**: Trading execution, order placement, account modifications, fiat payment methods, portfolio aggregation

## API Strategy

### Dual-API Architecture

Coinbase exposes two complementary APIs required for full portfolio visibility:

#### Coinbase Platform API (v2) — Account & Transaction Data
Base URL: `https://api.coinbase.com/v2/`

| Endpoint | Purpose | Maps To |
|----------|---------|---------|
| `GET /accounts` | List all currency accounts with balances | `Balance`, `Asset` |
| `GET /accounts/:id` | Single account detail | `Balance` |
| `GET /accounts/:id/transactions` | Sends, receives, buys, sells per account | `Transaction` |
| `GET /accounts/:id/buys` | Purchase history | `Transaction` (type: trade) |
| `GET /accounts/:id/sells` | Sale history | `Transaction` (type: trade) |
| `GET /accounts/:id/deposits` | Fiat/crypto deposits | `Transaction` (type: deposit) |
| `GET /accounts/:id/withdrawals` | Fiat/crypto withdrawals | `Transaction` (type: withdrawal) |
| `GET /exchange-rates` | Current exchange rates | `MarketData` |
| `GET /prices/:pair/spot` | Spot price for a trading pair | `MarketData` |
| `GET /currencies` | Supported currencies and metadata | Token metadata cache |

#### Coinbase Advanced Trade API (v3) — Trading & Portfolio Detail
Base URL: `https://api.coinbase.com/api/v3/brokerage/`

| Endpoint | Purpose | Maps To |
|----------|---------|---------|
| `GET /accounts` | Trading accounts with available/hold balances | `Balance` |
| `GET /portfolios` | Portfolio-level groupings | Portfolio metadata |
| `GET /orders/historical` | Full order history with fill detail | `Transaction` (type: trade) |
| `GET /orders/historical/:id` | Single order detail with fills | `Transaction` |
| `GET /transaction_summary` | Fee summary by volume tier | `CEXFeeSchedule` |
| `GET /products` | Available trading pairs | Market pair metadata |
| `GET /products/:id/candles` | Historical OHLCV candles | `PriceHistory` |
| `GET /products/:id/ticker` | Real-time best bid/ask | `MarketData` |

#### API Selection Strategy

- **v2 for account/balance data**: The Platform API provides the complete list of all accounts (including staking, vault, and earn) which v3 does not expose
- **v3 for trading data**: Advanced Trade API provides granular order fills, fee breakdowns, and OHLCV data that v2 lacks
- **v2 for transaction history**: Sends/receives/conversions are only available through v2
- **v3 for portfolio groupings**: Named portfolio support (Coinbase Advanced feature)

### WebSocket Feed (Advanced Trade)
URL: `wss://advanced-trade-ws.coinbase.com`

| Channel | Purpose | Domain Event |
|---------|---------|-------------|
| `user` | Order fills, account updates | `BalanceUpdated`, `TransactionDetected` |
| `ticker` | Real-time price updates | `PriceUpdated` |
| `heartbeats` | Connection health | Internal health check |

WebSocket is used for live updates when the application is active. Falls back to polling (60s interval) when WebSocket is unavailable, per the en-p0ha WebSocket-first directive.

### Rate Limits

| API | Limit | Strategy |
|-----|-------|----------|
| Platform API (v2) | 10,000 requests/hour | Token bucket, batch where possible |
| Advanced Trade (v3) | 30 requests/second | Token bucket with burst capacity |
| WebSocket | 750 messages/second per connection | Message coalescing |

## Authentication Strategy

### OAuth2 with PKCE (Recommended — Primary Path)

OAuth2 Authorization Code Flow with PKCE is the correct choice for a browser-first, client-side application. No client secret is required.

#### Authorization Flow

1. **Initiate**: Generate `code_verifier` (cryptographically random, 43-128 chars) and `code_challenge` (SHA-256 hash, base64url-encoded)
2. **Redirect**: Send user to `https://www.coinbase.com/oauth/authorize` with:
   - `response_type=code`
   - `client_id={app_client_id}`
   - `redirect_uri={app_callback_url}`
   - `scope=wallet:accounts:read,wallet:transactions:read,wallet:buys:read,wallet:sells:read,wallet:deposits:read,wallet:withdrawals:read`
   - `code_challenge={code_challenge}`
   - `code_challenge_method=S256`
   - `state={csrf_token}`
3. **Callback**: Exchange authorization code for tokens at `POST /oauth/token` with `code_verifier`
4. **Store**: Encrypt access token and refresh token in browser storage (IndexedDB, encrypted)
5. **Refresh**: Automatic token refresh before expiry (tokens expire in 2 hours)
6. **Revoke**: Clean revocation via `POST /oauth/revoke` on disconnect

#### OAuth Scopes Required

| Scope | Purpose |
|-------|---------|
| `wallet:accounts:read` | Read account balances and metadata |
| `wallet:transactions:read` | Read transaction history |
| `wallet:buys:read` | Read buy (purchase) history |
| `wallet:sells:read` | Read sell history |
| `wallet:deposits:read` | Read deposit history |
| `wallet:withdrawals:read` | Read withdrawal history |

All scopes are read-only. No write scopes are requested, enforcing the Integration Domain's strict read-only guarantee at the OAuth permission level.

#### Advanced Trade API Access via OAuth

The Advanced Trade API endpoints are accessible through the same OAuth2 token when the user has Advanced Trade enabled on their Coinbase account. No additional authentication step is needed — the v3 endpoints accept the same Bearer token.

### API Key with HMAC-SHA256 (Alternative Path)

For users who prefer not to use OAuth or when OAuth is unavailable:

1. **Setup**: User generates an API key pair in Coinbase settings (key + secret)
2. **Storage**: Both key and secret encrypted in IndexedDB using the application's encryption layer
3. **Signing**: Each request signed with HMAC-SHA256: `signature = HMAC-SHA256(secret, timestamp + method + path + body)`
4. **Headers**: `CB-ACCESS-KEY`, `CB-ACCESS-SIGN`, `CB-ACCESS-TIMESTAMP`

**Client-side safety**: The API key and secret never leave the browser. Signing happens locally. The key is created by the user with read-only permissions — CygnusWealth instructs users to grant only read permissions.

### Security Measures

- Encrypted token/key storage in IndexedDB (using the application's existing encryption layer)
- PKCE prevents authorization code interception
- CSRF protection via `state` parameter
- Token expiration handling with automatic refresh
- No credential storage — only encrypted tokens
- Read-only scope enforcement at OAuth level
- Revocation support on disconnect/logout

## Data Transformation

### Unified Model Mapping

#### Coinbase Accounts → Asset + Balance

| Coinbase Field | CygnusWealth Model | Notes |
|---------------|-------------------|-------|
| `account.currency.code` | `Asset.symbol` | Standardized ticker |
| `account.currency.name` | `Asset.name` | Display name |
| `account.balance.amount` | `Balance.quantity` | Decimal string → BigNumber |
| `account.balance.currency` | `Balance.currency` | ISO currency code |
| `account.native_balance.amount` | `Value.amount` | Fiat valuation |
| `account.native_balance.currency` | `Value.currency` | User's native fiat |
| `account.type` | `CEXAccountType` | wallet, vault, fiat |
| `account.currency.asset_id` | `Asset.id` | Coinbase asset UUID |

#### Coinbase Transactions → Transaction

| Coinbase Field | CygnusWealth Model | Mapping |
|---------------|-------------------|---------|
| `transaction.type` (send) | `TransactionType.transfer` | Outbound transfer |
| `transaction.type` (receive, faucet_payout) | `TransactionType.transfer` | Inbound transfer |
| `transaction.type` (buy) | `TransactionType.trade` | Fiat → crypto purchase |
| `transaction.type` (sell) | `TransactionType.trade` | Crypto → fiat sale |
| `transaction.type` (trade, exchange_deposit) | `TransactionType.swap` | Crypto-to-crypto conversion |
| `transaction.type` (staking_reward, inflation_reward) | `TransactionType.deposit` | Staking/earn reward |
| `transaction.amount` | `Transaction.amount` | Signed decimal |
| `transaction.native_amount` | `Transaction.fiatValue` | Native currency value |
| `transaction.created_at` | `Transaction.timestamp` | ISO 8601 |
| `transaction.network.hash` | `Transaction.hash` | On-chain tx hash (if send/receive) |
| `transaction.status` | `TransactionStatus` | pending/completed/failed/cancelled |

#### Advanced Trade Orders → Transaction

| Coinbase Field | CygnusWealth Model | Mapping |
|---------------|-------------------|---------|
| `order.side` (BUY/SELL) | `TransactionType.trade` | Trade direction in metadata |
| `order.product_id` | `Transaction.asset` + `Transaction.metadata.tradingPair` | e.g., "BTC-USD" |
| `order.filled_size` | `Transaction.amount` | Quantity filled |
| `order.average_filled_price` | `Transaction.metadata.price` | Average execution price |
| `order.total_fees` | `Transaction.metadata.fees` | Total fees paid |
| `order.created_time` | `Transaction.timestamp` | ISO 8601 |
| `order.order_type` | `Transaction.metadata.orderType` | market/limit/stop |
| `order.status` | `TransactionStatus` | FILLED → confirmed, CANCELLED → cancelled, PENDING → pending |

### CEX-Specific Metadata (TradFi Metadata Pattern)

Following the Robinhood integration pattern of enriched metadata:

- **Cost basis**: Calculated from buy orders (FIFO, LIFO, or specific-lot — user configurable)
- **Realized/unrealized gains**: Computed from trade history against current spot price
- **Staking APY**: Current annualized yield for staked assets
- **Earn rewards**: Cumulative rewards earned per asset
- **Fee totals**: Aggregate fees paid by asset and time period
- **Trading volume**: 30-day volume tier (affects fee schedule)

## New Types Required in Data Models

The following types should be added to `@cygnus-wealth/data-models` in the **Extended tier** (expedited RFC):

### CEXAccountType Enum
Discriminates between different Coinbase account types that map to different data fetching strategies:
- `spot` — Standard trading/holding account
- `vault` — Time-locked vault account (delayed withdrawals)
- `fiat` — Fiat currency account (USD, EUR, etc.)
- `staking` — Staking position account
- `earn` — Coinbase Earn rewards account

### CEXOrderDetail Interface
Captures order-level detail not present in the base `Transaction` type:
- `orderId` — Exchange order identifier
- `tradingPair` — Product ID (e.g., "BTC-USD")
- `side` — buy | sell
- `orderType` — market | limit | stop_limit | bracket
- `filledSize` — Quantity filled
- `averagePrice` — Average execution price
- `totalFees` — Total fees for this order
- `fills` — Individual fill records (partial fills)
- `settlementDate` — When the trade settled

### CEXFeeSchedule Interface
Captures the user's fee tier for cost projection:
- `makerFeeRate` — Current maker fee percentage
- `takerFeeRate` — Current taker fee percentage
- `thirtyDayVolume` — Trailing 30-day trading volume
- `pricingTier` — Volume-based tier name

### CEXStakingPosition Interface
Extends the existing concept of staking (used in Solana for native SOL staking) for CEX-managed staking:
- `asset` — The staked asset symbol
- `stakedAmount` — Amount currently staked
- `rewardsEarned` — Total rewards earned to date
- `currentAPY` — Current annualized percentage yield
- `lockupPeriod` — Unstaking lockup duration (if any)
- `canUnstake` — Whether immediate unstaking is available
- `validatorOrPool` — Staking pool/validator identifier

### IntegrationSource Update
Add `coinbase` to the existing `IntegrationSource` type alongside `evm`, `solana`, `robinhood`.

## Integration Patterns

### Service Interface

The domain exposes services following the same pattern as Robinhood Integration:

#### Authentication Service
- Handle OAuth2 PKCE authorization flow
- Manage token storage and refresh
- Support API Key alternative flow
- Secure logout with token revocation
- Connection status reporting

#### Account Service
- Fetch all accounts (v2) with balances
- Fetch trading accounts (v3) with available/hold breakdown
- Aggregate across account types (spot, vault, staking, earn, fiat)
- Portfolio grouping support (v3 named portfolios)

#### Market Data Service
- Spot prices via v2 `/prices/:pair/spot`
- Real-time ticker via v3 `/products/:id/ticker`
- Historical OHLCV candles via v3 `/products/:id/candles`
- Exchange rates for fiat conversion

#### Transaction Service
- Transaction history via v2 (sends, receives, buys, sells, conversions)
- Order history via v3 (fills, partial fills, fee breakdown)
- Staking reward history
- Pagination with cursor-based traversal (Coinbase uses cursor pagination)
- Deduplication between v2 and v3 transaction records

### Anti-Corruption Layer

```
Coinbase API (v2 + v3) → CoinbaseAdapter → CoinbaseTranslator → CoinbaseValidator → Domain Models
```

#### CoinbaseAdapter
- Manages dual-API communication (v2 and v3 endpoints)
- Handles OAuth2 Bearer token injection and API key HMAC signing
- Request/response logging with PII redaction
- Rate limit tracking per API tier

#### CoinbaseTranslator
- Converts Coinbase account structures to unified `Asset` / `Balance`
- Maps Coinbase transaction types to `TransactionType` enum
- Normalizes amounts (Coinbase uses string decimals) to `BigNumber`
- Merges v2 and v3 data for trades (v2 provides basic info, v3 provides fill detail)
- Resolves Coinbase asset IDs to CygnusWealth standard symbols

#### CoinbaseValidator
- Validates balance non-negativity
- Validates transaction completeness (required fields present)
- Verifies timestamp ordering in transaction lists
- Sanitizes string fields against XSS
- Rejects malformed API responses

### Factory Function

Following the en-25w5 directive pattern used by EVM and Solana integrations:

```
createCoinbaseIntegration(config: CoinbaseIntegrationConfig): CoinbaseIntegrationService
```

`CoinbaseIntegrationConfig` contains:
- `authMethod` — `'oauth2'` | `'api_key'`
- `oauthConfig` — client ID, redirect URI, scopes (when authMethod is oauth2)
- `encryptionProvider` — interface for encrypting/decrypting stored tokens
- `cacheConfig` — TTLs and storage backend
- `rateLimitConfig` — per-API rate limit budgets

Note: Unlike EVM/Solana integrations, Coinbase integration does not use `RpcProviderConfig` since it communicates with Coinbase's centralized API, not blockchain RPC nodes. No fallback chain is needed — Coinbase's API is the single source of truth. The circuit breaker and retry patterns still apply to the Coinbase API endpoints.

## Error Handling

### Common Error Scenarios
- OAuth token expired (401)
- OAuth token revoked by user in Coinbase settings
- Rate limiting (429) — per-API tier
- API maintenance windows (503)
- Invalid/delisted trading pairs
- Account access permissions changed
- Network connectivity failures

### Recovery Strategies
1. **Token Expiry (401)**: Automatic refresh via refresh token. If refresh fails, prompt re-authorization
2. **Token Revoked**: Detect on 401 after refresh failure, prompt re-authorization, clear stored tokens
3. **Rate Limiting (429)**: Exponential backoff, respect `Retry-After` header, coordinate across v2/v3
4. **API Downtime (503)**: Cached data fallback, surface stale data indicator to user
5. **Network Issues**: Retry with exponential backoff + jitter, fall back to cached data
6. **Permission Changes**: Detect reduced scope on API calls, prompt re-authorization with required scopes

### Domain Events

| Event | Trigger | Payload |
|-------|---------|---------|
| `CoinbaseConnected` | OAuth flow completed | Account count, scopes granted |
| `CoinbaseDisconnected` | User revokes or disconnects | Account ID |
| `CoinbaseBalanceUpdated` | Account balance change detected | Account ID, new balance |
| `CoinbaseTransactionDetected` | New transaction or order fill | Transaction detail |
| `CoinbaseAuthExpired` | Token refresh failed | Reason, re-auth required flag |
| `CoinbaseRateLimited` | 429 received | API tier, retry-after duration |

## Caching Strategy

### Cache Layers
Following the domain-wide L1/L2/L3 pattern:

| Data Type | L1 (Memory) TTL | L2 (IndexedDB) TTL | Notes |
|-----------|-----------------|---------------------|-------|
| Account list + balances | 60s | 5min | Primary portfolio data |
| Spot prices | 30s | 60s | Frequently accessed |
| Transaction history | 5min | 30min | Paginated, append-only |
| Order history | 5min | 30min | Paginated, append-only |
| Staking positions | 5min | 15min | Infrequently changes |
| Currency metadata | 24h | 7d | Rarely changes |
| Exchange rates | 30s | 60s | Used for fiat conversion |
| Fee schedule | 1h | 24h | Changes with volume tier |

### Cache Invalidation
- On WebSocket `user` channel account update
- After successful OAuth re-authorization (full refresh)
- Manual refresh trigger from user
- On detection of new transaction via polling

## Rate Limiting

### Budget Allocation

The Coinbase integration manages two independent rate limit budgets:

| API | Budget | Allocation |
|-----|--------|------------|
| Platform v2 (10k/hr) | 8,000/hr for data fetching, 2,000/hr reserved for auth and metadata | Token bucket, 2.2 req/s sustained |
| Advanced Trade v3 (30/s) | 25/s for data fetching, 5/s reserved for burst | Token bucket with burst capacity |

### Optimization Strategies
- Batch account balance fetching (v2 list endpoint returns all accounts)
- Cursor-based pagination with reasonable page sizes (100 items)
- Prefer v2 list endpoints over individual account queries
- Cache currency metadata aggressively (24h TTL)
- Use WebSocket for real-time updates instead of polling when available
- Coordinate v2 and v3 requests to avoid redundant data fetching

## Testing Strategy

### Test Coverage Areas
- OAuth2 PKCE flow (authorization, token exchange, refresh, revocation)
- API Key HMAC signing correctness
- Dual-API data merging (v2 + v3 trade reconciliation)
- Data transformation accuracy (all Coinbase types → CygnusWealth models)
- Rate limit compliance across both API tiers
- Error recovery for all documented scenarios
- Cache effectiveness and invalidation
- WebSocket reconnection and fallback to polling

### Integration Testing
- Full account sync with mock Coinbase API
- Transaction history pagination traversal
- Multi-account aggregation (spot + vault + staking + earn)
- OAuth token lifecycle (grant → refresh → expire → re-auth)
- Stale data detection and freshness indicators

### Test Distribution (per domain standard)
- **Unit Tests (80%)**: Translator, Validator, rate limiter, HMAC signing, PKCE generation
- **Integration Tests (15%)**: Full fetch flows with mocked API, cache layer interaction
- **Contract Tests (4%)**: Data model compliance, event schema validation
- **E2E Tests (1%)**: Critical path — OAuth connect → fetch balances → display in portfolio

## Configuration

### Configurable Parameters
- API base URLs (for future Coinbase API versioning)
- Request timeouts (default: 10s for v2, 5s for v3)
- Retry attempts (default: 3)
- Cache durations (overridable per data type)
- Rate limit thresholds
- WebSocket reconnection parameters
- Page sizes for pagination (default: 100)

### Environment Settings
Following the domain convention where only `cygnus-wealth-app` reads env vars:

| Variable | Purpose |
|----------|---------|
| `CYGNUS_COINBASE_CLIENT_ID` | OAuth2 application client ID |
| `CYGNUS_COINBASE_REDIRECT_URI` | OAuth2 callback URL |

Note: No API secrets in environment variables. The OAuth2 PKCE flow requires no client secret. For the API Key path, keys are user-provided and stored encrypted in browser storage, never in environment variables.

## Compliance Considerations

### Regulatory Requirements
- Read-only access enforcement (no write scopes, no trading endpoints called)
- Data privacy compliance (all data stored locally in browser, encrypted)
- Audit trail maintenance (structured logging of all API interactions)
- Coinbase Terms of Service adherence (official API, not scraping)

### Data Handling
- No storage of OAuth client secrets (PKCE eliminates the need)
- Encrypted token storage in IndexedDB
- PII data minimization (only fetch what's needed for portfolio display)
- Secure data transmission (HTTPS only, certificate pinning where supported)
- Token cleanup on disconnect

## Performance Targets

- OAuth authorization flow: Under 5 seconds (includes redirect)
- Account/balance fetch: Under 2 seconds (all accounts)
- Spot price fetch: Under 500ms (cached), under 1.5s (fresh)
- Transaction history: Under 2 seconds for 100 items
- Full portfolio data: Under 3 seconds (accounts + balances + staking)
- Cache hit ratio: Above 70%
- WebSocket reconnection: Under 3 seconds

## Rig Structure

### Repository
`@cygnus-wealth/coinbase-integration`

### Source Layout

```
coinbase-integration/
├── src/
│   ├── index.ts                      # Public API: createCoinbaseIntegration()
│   ├── types.ts                      # CoinbaseIntegrationConfig, internal types
│   ├── auth/
│   │   ├── oauth2-pkce.ts            # OAuth2 PKCE flow implementation
│   │   ├── api-key-hmac.ts           # API Key HMAC-SHA256 signing
│   │   └── token-manager.ts          # Token storage, refresh, revocation
│   ├── adapters/
│   │   ├── platform-api-adapter.ts   # Coinbase Platform API v2 client
│   │   ├── advanced-trade-adapter.ts # Coinbase Advanced Trade API v3 client
│   │   └── websocket-adapter.ts      # Advanced Trade WebSocket client
│   ├── translators/
│   │   ├── account-translator.ts     # Account/balance → Asset/Balance
│   │   ├── transaction-translator.ts # Transactions → Transaction model
│   │   ├── order-translator.ts       # Orders → Transaction model with CEXOrderDetail
│   │   └── market-data-translator.ts # Prices/candles → MarketData/PriceHistory
│   ├── validators/
│   │   ├── balance-validator.ts      # Balance data integrity
│   │   ├── transaction-validator.ts  # Transaction completeness
│   │   └── response-validator.ts     # API response schema validation
│   ├── services/
│   │   ├── authentication-service.ts # Auth orchestration (OAuth + API Key)
│   │   ├── account-service.ts        # Account + balance aggregation
│   │   ├── transaction-service.ts    # Transaction + order history
│   │   ├── market-data-service.ts    # Prices, candles, exchange rates
│   │   └── subscription-service.ts   # WebSocket live updates (en-p0ha)
│   └── infrastructure/
│       ├── rate-limiter.ts           # Dual-tier rate limit management
│       ├── circuit-breaker.ts        # Per-endpoint circuit breaker
│       └── cache-manager.ts          # L1/L2 cache with TTL strategy
├── tests/
│   ├── unit/
│   │   ├── auth/
│   │   ├── translators/
│   │   ├── validators/
│   │   └── infrastructure/
│   ├── integration/
│   │   ├── account-sync.test.ts
│   │   ├── transaction-fetch.test.ts
│   │   └── oauth-lifecycle.test.ts
│   └── fixtures/
│       ├── coinbase-accounts.json
│       ├── coinbase-transactions.json
│       └── coinbase-orders.json
├── package.json
├── tsconfig.json
├── vitest.config.ts
└── README.md
```

### Dependencies
- **Internal**: `@cygnus-wealth/data-models` (Contract Domain)
- **External**: None beyond standard Web APIs (fetch, crypto.subtle for HMAC/SHA-256, WebSocket)
- **No ethers.js or @solana/web3.js** — this integration communicates with a centralized API, not blockchain RPCs

## Limitations

### API Restrictions
- Read-only access (by design and OAuth scope)
- No access to Coinbase Pro (deprecated, migrated to Advanced Trade)
- Historical data limited by Coinbase's retention policies
- Staking data availability varies by asset and region
- Some Coinbase Earn programs may not expose detailed position data via API

### Data Availability
- Real-time balance updates require WebSocket connection
- Transaction history pagination may be slow for accounts with thousands of transactions
- Currency metadata may lag for newly listed assets
- Fiat account balances subject to banking/settlement delays
- Coinbase regional availability affects which features are accessible

## Future Enhancements

- Coinbase Wallet (self-custody) integration via WalletConnect
- Coinbase Commerce payment tracking
- Tax lot reporting integration (FIFO/LIFO/HIFO cost basis methods)
- Advanced portfolio analytics (P&L curves, trade performance)
- Coinbase NFT marketplace position tracking
- Multi-exchange order book aggregation (when Kraken/Binance integrations exist)

---

## Related Documentation

- **[← Integration Domain README](../README.md)** - Domain overview and strategic guidance
- **[Integration Patterns](../patterns.md)** - Apply domain patterns to this context
- **[Resilience & Performance](../resilience-performance.md)** - Implement resilience strategies
- **[Testing & Security](../testing-security.md)** - Apply testing and security standards
- **[Robinhood Integration](./robinhood-integration.md)** - Sibling CEX integration (pattern reference)
- **Other Bounded Contexts**: [Wallet Integration System](./wallet-integration-system.md) | [EVM Integration](./evm-integration.md) | [Solana Integration](./sol-integration.md)
