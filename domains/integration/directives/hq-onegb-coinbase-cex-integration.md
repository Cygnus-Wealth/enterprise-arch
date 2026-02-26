# hq-onegb: Coinbase CEX Integration Architecture

> **Directive**: Define the Coinbase centralized exchange integration as a new bounded context within the Integration Domain.
>
> **Status**: Architecture specification complete
>
> **Artifacts Produced**: `domains/integration/bounded-contexts/coinbase-integration.md`

## Executive Summary

This directive specifies the architecture for `@cygnus-wealth/coinbase-integration`, a new bounded context within the Integration Domain. It provides read-only access to Coinbase accounts via the official Coinbase Platform API (v2) and Advanced Trade API (v3), enabling users to include their CEX holdings in their aggregated CygnusWealth portfolio.

The integration follows the established patterns from the Robinhood Integration while addressing Coinbase-specific requirements: dual-API architecture, OAuth2 with PKCE for client-side safety, staking/earn position tracking, and WebSocket live updates.

## Architecture Decisions

### 1. Dual-API Strategy

**Decision**: Use both Coinbase Platform API (v2) and Advanced Trade API (v3) in combination.

**Rationale**: Neither API alone provides full portfolio visibility. v2 exposes all account types (spot, vault, staking, earn, fiat) and transaction history (sends/receives/conversions). v3 exposes granular order fill data, fee breakdowns, named portfolios, and OHLCV candles. Using both yields complete data coverage.

**Key endpoints**:
- v2 `GET /accounts` — All accounts with balances (including staking, vault, earn)
- v2 `GET /accounts/:id/transactions` — Sends, receives, buys, sells, conversions
- v3 `GET /accounts` — Trading accounts with available/hold breakdown
- v3 `GET /orders/historical` — Order history with individual fills and fee detail
- v3 `GET /transaction_summary` — Fee tier information
- v3 `GET /products/:id/ticker` — Real-time prices
- v3 WebSocket `user` channel — Live account updates

### 2. OAuth2 with PKCE (Primary Authentication)

**Decision**: OAuth2 Authorization Code Flow with PKCE as the primary authentication method, with API Key + HMAC-SHA256 as an alternative path.

**Rationale**: PKCE eliminates the need for a client secret, making it safe for browser-first applications. The user authorizes via Coinbase's consent screen, granting read-only scopes. Tokens are stored encrypted in IndexedDB, automatically refreshed, and revocable. This aligns with CygnusWealth's privacy-first, client-side sovereignty principles.

**Read-only scopes**: `wallet:accounts:read`, `wallet:transactions:read`, `wallet:buys:read`, `wallet:sells:read`, `wallet:deposits:read`, `wallet:withdrawals:read`. No write scopes requested.

**API Key alternative**: For users who prefer manual key management. User generates read-only API key in Coinbase settings, provides key+secret to CygnusWealth, stored encrypted locally, requests signed with HMAC-SHA256. Keys never leave the browser.

### 3. Data Model Mappings

**Decision**: Map Coinbase data to existing `Asset`, `Balance`, `Transaction`, `MarketData`, `PriceHistory` types from `@cygnus-wealth/data-models`.

**Mapping summary**:
- Coinbase accounts → `Asset` (per currency) + `Balance` (quantity + native fiat value)
- Coinbase transactions (sends/receives/buys/sells) → `Transaction` with appropriate `TransactionType`
- Coinbase orders (v3) → `Transaction` (type: trade) enriched with `CEXOrderDetail` metadata
- Coinbase spot prices → `MarketData`
- Coinbase candles → `PriceHistory`
- Staking positions → `CEXStakingPosition` (new type)

**IntegrationSource**: Add `coinbase` to the existing enum alongside `evm`, `solana`, `robinhood`.

### 4. New Types for Data Models (Extended Tier)

**Decision**: Add four new types to `@cygnus-wealth/data-models` under the Extended stability tier (expedited RFC).

| Type | Purpose |
|------|---------|
| `CEXAccountType` | Enum: `spot`, `vault`, `fiat`, `staking`, `earn` — discriminates account types |
| `CEXOrderDetail` | Order-level metadata: trading pair, side, fills, fees, settlement |
| `CEXFeeSchedule` | User's fee tier: maker/taker rates, 30-day volume, tier name |
| `CEXStakingPosition` | CEX-managed staking: asset, amount, rewards, APY, lockup |

These types are CEX-generic (not Coinbase-specific) to support future Kraken and Binance integrations.

### 5. Rig Structure

**Decision**: Follow the established integration pattern with a factory function entry point.

**Repository**: `@cygnus-wealth/coinbase-integration`

**Entry point**: `createCoinbaseIntegration(config: CoinbaseIntegrationConfig): CoinbaseIntegrationService`

**Internal structure**: `auth/` (OAuth2 PKCE + API Key HMAC), `adapters/` (v2 + v3 + WebSocket clients), `translators/` (account, transaction, order, market data mapping), `validators/` (balance, transaction, response schema), `services/` (authentication, account, transaction, market data, subscription), `infrastructure/` (rate limiter, circuit breaker, cache manager).

**Dependencies**: Only `@cygnus-wealth/data-models`. No blockchain libraries needed — this integration communicates with Coinbase's centralized REST/WebSocket API.

### 6. Resilience and Performance

**Decision**: Apply domain-standard patterns with Coinbase-specific configuration.

- **Circuit breaker**: Per-endpoint (v2 and v3 tracked independently)
- **Rate limiting**: Dual-tier token bucket (v2: 10k/hr, v3: 30/s)
- **Caching**: L1 memory + L2 IndexedDB with data-type-specific TTLs (balances 60s, transactions 5min, metadata 24h)
- **WebSocket**: Live updates via Advanced Trade WebSocket `user` channel, polling fallback at 60s
- **Performance targets**: Account fetch < 2s, price fetch < 500ms cached, full portfolio < 3s

## Contract Updates Required

### BlockchainIntegrationContract Extension

The existing `BlockchainIntegrationContract` (Portfolio Aggregation → Integration) needs a parallel `CEXIntegrationContract`:

```
CEXIntegrationContract {
  getAccounts(): CEXAccountList
  getBalances(): BalanceList
  getTransactions(options): TransactionList
  getStakingPositions(): StakingPositionList
  subscribeToUpdates(): Subscription
}
```

This is a new contract type — CEX integrations do not use `getTokens()`, `getNFTs()`, or address-based queries. Instead they use account-based queries authenticated via OAuth/API key.

### Portfolio Aggregation Impact

Portfolio Aggregation must be updated to:
1. Register Coinbase as an integration source
2. Call `CEXIntegrationContract` alongside `BlockchainIntegrationContract`
3. Merge CEX balances into the unified portfolio with proper deduplication
4. Apply the "on-chain data preferred over CEX data" reconciliation rule for assets that appear in both (e.g., if a user's ETH shows up both on-chain via wallet and in Coinbase, avoid double-counting)

### Data Models Impact

Add to `@cygnus-wealth/data-models`:
1. `CEXAccountType` enum (Extended tier)
2. `CEXOrderDetail` interface (Extended tier)
3. `CEXFeeSchedule` interface (Extended tier)
4. `CEXStakingPosition` interface (Extended tier)
5. Add `'coinbase'` to `IntegrationSource` enum (Standard tier — existing enum update)

## Implementation Phases

**Phase 1 — Foundation**: OAuth2 PKCE flow, token management, Platform API v2 adapter, account/balance fetching, basic data transformation, unit test infrastructure

**Phase 2 — Core Features**: Transaction history (v2), order history (v3), data merging (v2+v3 trade reconciliation), market data service, caching layer, integration tests

**Phase 3 — Resilience**: Circuit breakers, rate limit management, error recovery flows, stale data indicators, connection status reporting

**Phase 4 — Enhancement**: WebSocket subscription service, staking/earn position tracking, API Key alternative auth path, advanced caching (warming, predictive), E2E tests

## Full Specification

The complete bounded context specification is at: **[domains/integration/bounded-contexts/coinbase-integration.md](../domains/integration/bounded-contexts/coinbase-integration.md)**
