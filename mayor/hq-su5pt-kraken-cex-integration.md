# hq-su5pt: Kraken CEX Integration — Bounded Context Architecture

**Status**: RECOMMENDATION
**Date**: 2026-02-18
**Bead**: hq-su5pt
**Scope**: Integration Domain, Kraken bounded context, DataModels extensions, CEX credential management, portfolio aggregation contract

---

## Executive Summary

**Kraken is the next centralized exchange integration for CygnusWealth, following the Robinhood pattern but adapted for crypto-native CEX specifics.**

The Integration Domain's "Future Expansion" section lists Kraken as a planned CEX integration. This directive defines the bounded context architecture for `kraken-integration`: a read-only, client-side TypeScript library that fetches account balances, trade history, open positions, and ledger entries from Kraken's Spot REST API, transforms them into CygnusWealth's unified data models, and exposes a service facade for consumption by portfolio-aggregation. The integration follows the layered architecture established by `robinhood-integration` (Transport → API → Mapper → Service) while addressing Kraken-specific concerns: HMAC-SHA512 authentication with nonce management, Kraken's counter-based rate limiting, asset pair naming conventions (e.g., `XXBTZUSD`), and margin/staking position types that map to existing DataModels account and position types.

---

## 1. Bounded Context Definition

### Identity

- **Name**: Kraken Integration
- **Repository**: `kraken-integration`
- **Package**: `@cygnus-wealth/kraken-integration`
- **Domain**: Integration
- **GitHub Topics**: `domain-integration`, `context-kraken`, `type-library`, `cex`, `typescript`

### Core Responsibilities

1. **Authentication**: HMAC-SHA512 signed requests with nonce management and optional 2FA/OTP
2. **Balance Retrieval**: Fetch spot, margin, and staking balances via Kraken private endpoints
3. **Trade History**: Paginated retrieval of fills/trades with offset-based pagination
4. **Position Tracking**: Open margin positions, staking positions
5. **Ledger Access**: Account ledger entries (deposits, withdrawals, trades, staking rewards)
6. **Data Transformation**: Convert Kraken API responses to CygnusWealth unified data models

### Domain Boundaries

- **Owns**: Kraken API interactions, HMAC signature generation, nonce lifecycle, Kraken-specific data transformation, asset pair name resolution
- **Does NOT Own**: Trading execution, order placement/cancellation, deposit/withdrawal initiation, portfolio aggregation, asset pricing

### Read-Only Guarantee

The integration uses only Kraken API endpoints that require `Query` permission. No endpoints requiring `Trade`, `Deposit`, or `Withdraw` permissions are invoked. The API key permission model enforces this at the Kraken side — users create a read-only API key with only "Query Funds" and "Query Open Orders & Trades" and "Query Closed Orders & Trades" permissions.

---

## 2. API Strategy

### Base Configuration

- **Base URL**: `https://api.kraken.com`
- **Protocol**: HTTPS REST (POST for all private endpoints)
- **Content Type**: `application/x-www-form-urlencoded` (Kraken requires form-encoded POST bodies for private endpoints)
- **Public Endpoints Prefix**: `/0/public/`
- **Private Endpoints Prefix**: `/0/private/`

### Endpoints Used

#### Account Data (Private — Read-Only)

| Endpoint | Path | Counter Cost | Purpose | CygnusWealth Mapping |
|----------|------|-------------|---------|---------------------|
| Get Account Balance | `/0/private/Balance` | +1 | Spot balances per asset | `Balance[]` per `Account(type=SPOT)` |
| Get Extended Balance | `/0/private/BalanceEx` | +1 | Available, held, and credit amounts | Enriches `Balance` with held/available breakdown |
| Get Trade Balance | `/0/private/TradeBalance` | +1 | Margin equity, free margin, cost basis | `Account(type=MARGIN)` with margin-specific metadata |
| Get Open Positions | `/0/private/OpenPositions` | +1 | Margin positions (long/short) | `Balance[]` with position metadata |
| Get Trades History | `/0/private/TradesHistory` | +2 | Historical fills (50 per page) | `Transaction[]` |
| Get Ledgers Info | `/0/private/Ledgers` | +2 | Deposits, withdrawals, trades, staking | `Transaction[]` (wider scope than trades) |
| Get Closed Orders | `/0/private/ClosedOrders` | +1 | Completed/cancelled orders | `Transaction[]` (order-level view) |

#### Market Data (Public — No Auth)

| Endpoint | Path | Purpose | CygnusWealth Mapping |
|----------|------|---------|---------------------|
| Get Asset Info | `/0/public/Assets` | Asset metadata (decimals, display name) | `Asset` enrichment |
| Get Tradable Asset Pairs | `/0/public/AssetPairs` | Pair naming, precision, fees | Pair resolution for trade mapping |
| Get Ticker Information | `/0/public/Ticker` | Current price, volume, OHLC | `MarketData` |

#### Staking (Private — Read-Only)

| Endpoint | Path | Counter Cost | Purpose | CygnusWealth Mapping |
|----------|------|-------------|---------|---------------------|
| List Staking Transactions | `/0/private/Staking/Transactions` | +2 | Staking/unstaking history | `Transaction(type=STAKE\|UNSTAKE\|CLAIM_REWARD)` |
| List Stakeable Assets | `/0/private/Staking/Assets` | +1 | Available staking options with APY | `StakedPosition` enrichment |
| Get Staking Pending | `/0/private/Staking/Pending` | +1 | Pending stake/unstake operations | `StakedPosition` status tracking |

### Pagination Strategy

Kraken uses offset-based pagination for history endpoints:

- **TradesHistory**: Returns 50 results per call. Pass `ofs` (offset) parameter for pagination. Response includes `count` field for total result count.
- **Ledgers**: Same pattern — 50 results, `ofs` offset, `count` total.
- **ClosedOrders**: Same pattern — 50 results, `ofs` offset, `count` total.

The API client implements an auto-paginator that fetches all pages sequentially, respecting rate limits between pages (counter cost of +2 per history call requires pacing).

### Kraken Asset Naming

Kraken uses non-standard asset names with `X` and `Z` prefixes for legacy assets:

| Kraken Name | Standard Symbol | Type |
|-------------|----------------|------|
| `XXBT` | BTC | Crypto |
| `XETH` | ETH | Crypto |
| `XXRP` | XRP | Crypto |
| `ZUSD` | USD | Fiat |
| `ZEUR` | EUR | Fiat |
| `ZGBP` | GBP | Fiat |

Newer assets use standard tickers (e.g., `SOL`, `DOT`, `MATIC`). The translator layer maintains a mapping table sourced from the `/0/public/Assets` endpoint's `altname` field, which provides the canonical symbol for each Kraken-internal name.

---

## 3. Authentication Pattern

### Signature Algorithm

Kraken private endpoints require two custom HTTP headers:

- **`API-Key`**: The public API key string
- **`API-Sign`**: HMAC-SHA512 signature, base64-encoded

**Signature computation**:

```
API-Sign = Base64(HMAC-SHA512(
  key: Base64Decode(apiSecret),
  message: uriPath + SHA256(nonce + postData)
))
```

Where:
- `uriPath` = e.g., `/0/private/Balance`
- `nonce` = monotonically increasing 64-bit integer (typically `Date.now()`)
- `postData` = URL-encoded form body including `nonce=...` and any other parameters

### Nonce Management

The nonce must be strictly increasing per API key across all requests. The implementation uses `Date.now()` as the base, with a monotonic guard:

- Track `lastNonce` in-memory
- Each request: `nonce = Math.max(Date.now(), lastNonce + 1)`
- Store `lastNonce = nonce`

This prevents nonce collisions from rapid sequential requests within the same millisecond.

### Client-Side Safety Analysis

**API Key + Secret are provided by the user and stored client-side.** This is the same trust model as the Robinhood integration's credential handling.

**Security properties**:
- The API secret never leaves the browser — HMAC signing happens in JavaScript using the Web Crypto API (`crypto.subtle`) or a pure-JS HMAC implementation
- Kraken supports permission-scoped API keys — the integration only requires `Query` permissions. Even if the key were compromised, no funds could be moved
- The nonce prevents replay attacks — each signed request is unique
- No CORS concerns — Kraken's API supports direct browser requests for private endpoints (unlike some exchanges)
- API keys can be further restricted by IP whitelist on Kraken's side

**Credential storage**:
- Uses `IntegrationCredentials` from DataModels with `source: IntegrationSource.KRAKEN`
- Fields used: `apiKey` (public key), `apiSecret` (private key for signing)
- `passphrase` field not used (Kraken does not use passphrases, unlike Coinbase)
- Credentials stored via the same encrypted storage mechanism used by Robinhood integration

### Optional 2FA (OTP)

If the user has 2FA enabled on their Kraken API key, the `otp` parameter must be included in the POST body. The service prompts for this during initial authentication verification and includes it in all subsequent requests for the session if required.

Detection: The first API call returns error `EAPI:Invalid key` or a 2FA challenge response. The service catches this and emits an event requesting OTP input from the user.

---

## 4. Data Model Mappings

### Mapping to Existing DataModels Types

#### Account Mapping

| Kraken Concept | DataModels Type | Field Mapping |
|----------------|----------------|---------------|
| Spot account | `Account` | `type: AccountType.SPOT, source: IntegrationSource.KRAKEN` |
| Margin account | `Account` | `type: AccountType.MARGIN, source: IntegrationSource.KRAKEN` |
| Staking | `Account` | `type: AccountType.STAKING, source: IntegrationSource.KRAKEN` |

Kraken does not have explicit "account IDs" — it's a single account with different balance views. The integration creates virtual account partitions (spot, margin, staking) under a single `Portfolio` for the user's Kraken connection.

#### Balance Mapping

| Kraken Field | DataModels `Balance` Field | Notes |
|-------------|---------------------------|-------|
| Asset balance (from `/Balance`) | `amount` | String for precision preservation |
| Asset symbol (resolved via translator) | `asset.symbol` | After Kraken name → standard symbol translation |
| Asset decimals (from `/Assets`) | `asset.decimals` | Kraken provides `decimals` per asset |
| Extended balance `balance` | `amount` | Total balance |
| Extended balance `hold_trade` | `metadata["kraken:holdTrade"]` | Amount held in open orders |
| Extended balance `credit` | `metadata["kraken:credit"]` | Credit line amount (if applicable) |

#### Transaction Mapping

| Kraken TradesHistory Field | DataModels `Transaction` Field | Notes |
|---------------------------|-------------------------------|-------|
| `pair` | Resolved to `assetsIn[0].asset` + `assetsOut[0].asset` | Pair split via AssetPairs lookup |
| `type` (buy/sell) | `type: TransactionType.BUY \| SELL` | Direct mapping |
| `price` | `assetsOut[0].value` or `assetsIn[0].value` | Depends on buy/sell direction |
| `vol` | `assetsIn[0].amount` or `assetsOut[0].amount` | Volume of the base asset |
| `cost` | Counter-asset amount | `price × vol` |
| `fee` | `fees[0].amount` | Fee in the counter asset |
| `time` | `timestamp` | Unix timestamp → ISO string |
| `ordertxid` | `metadata["kraken:orderTxId"]` | Reference to parent order |
| `trade_id` | `id` | Unique trade identifier |

#### Ledger → Transaction Mapping

| Kraken Ledger `type` | DataModels `TransactionType` | Notes |
|----------------------|-------------------------------|-------|
| `deposit` | `TRANSFER_IN` | Incoming funds |
| `withdrawal` | `TRANSFER_OUT` | Outgoing funds |
| `trade` | `BUY` or `SELL` | Determined by amount sign |
| `margin` | `BUY` or `SELL` | Margin trade |
| `staking` | `STAKE` | Staking lock |
| `reward` | `CLAIM_REWARD` | Staking reward |
| `transfer` | `TRANSFER_IN` or `TRANSFER_OUT` | Internal sub-account transfer |

#### Open Position Mapping

| Kraken OpenPositions Field | DataModels Mapping | Notes |
|---------------------------|-------------------|-------|
| `pair` | Balance `asset` (resolved) | Position's trading pair |
| `type` (buy/sell) | `metadata["kraken:positionSide"]` | Long (buy) or Short (sell) |
| `vol` | `Balance.amount` | Position size |
| `cost` | `Balance.cost_basis` | Entry cost |
| `fee` | `metadata["kraken:fee"]` | Fees paid |
| `net` | `Balance.unrealized_pnl` | Net P&L |
| `margin` | `metadata["kraken:margin"]` | Margin used |

#### Staking Mapping

| Kraken Staking Concept | DataModels Type | Notes |
|-----------------------|----------------|-------|
| Staked asset | `StakedPosition` | Uses existing StakedPosition interface |
| Staking method | `StakedPosition.protocol` | `"kraken"` (centralized staking) |
| APY/APR | `StakedPosition.apr` | From Staking/Assets endpoint |
| Reward | `StakedPosition.rewards[]` | Balance array of earned rewards |
| Lock period | `StakedPosition.lockupPeriod` | From Staking/Assets `minimum_amount.lock` |
| Unbonding | `StakedPosition.unlockDate` | Calculated from pending unstake |

---

## 5. New Types Needed in DataModels

### Assessment: Minimal New Types Required

The existing DataModels architecture handles Kraken well via the metadata extension pattern. The following types are **not** needed as new interfaces because they map cleanly to existing types:

- Spot balances → `Balance[]` on `Account(type=SPOT)`
- Trade history → `Transaction[]` with `TransactionType.BUY/SELL`
- Staking → `StakedPosition` (already exists)
- Margin positions → `Balance[]` on `Account(type=MARGIN)` with metadata

### Recommended DataModels Additions

#### 1. IntegrationSource Enum — Already Present

`IntegrationSource.KRAKEN` is already defined in the DataModels enum. No change needed.

#### 2. New Enum Value: AccountType.STAKING — Already Present

`AccountType.STAKING` is already defined. No change needed.

#### 3. CEX Fee Schedule Type (New — Recommended)

As more CEX integrations are added (Coinbase, Binance after Kraken), a shared fee schedule type will be reused. This is the one genuinely new type worth adding now:

```
Interface: CexFeeSchedule
Fields:
  - makerFee: string          // Maker fee rate (e.g., "0.0016" for 0.16%)
  - takerFee: string          // Taker fee rate (e.g., "0.0026" for 0.26%)
  - volume30d: string         // 30-day volume in USD
  - currency: string          // Fee currency
  - tierLevel: string         // User's fee tier name
  - nextTierVolume?: string   // Volume needed for next tier
  - nextTierMakerFee?: string // Maker fee at next tier
  - nextTierTakerFee?: string // Taker fee at next tier
```

This maps from Kraken's `/0/private/TradeVolume` response and will be reusable for Coinbase and Binance fee schedules.

**Location**: `DataModels/mayor/rig/src/interfaces/cex-fee-schedule.ts`

#### 4. Metadata Namespace Convention for Kraken

All Kraken-specific data stored in metadata fields follows the existing namespace convention:

```
"kraken:holdTrade"         // Amount held in open orders
"kraken:credit"            // Credit line amount
"kraken:positionSide"      // "long" | "short" for margin positions
"kraken:margin"            // Margin allocated to position
"kraken:fee"               // Position-specific fee
"kraken:orderTxId"         // Parent order reference
"kraken:leverage"          // Position leverage ratio
"kraken:assetClass"        // Kraken asset class (currency, forex)
"kraken:stakingMethod"     // On-chain vs off-chain staking
"kraken:unbondingDays"     // Days until unstake completes
```

No new interface needed — this uses the existing `Metadata` extension object pattern (`{[key: string]: unknown}`).

---

## 6. Rig Structure

### Package Layout

Following the established pattern from `robinhood-integration`:

```
kraken-integration/
├── refinery/rig/
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.ts              // HTTP transport, HMAC signing, nonce mgmt
│   │   │   ├── endpoints.ts           // Endpoint path constants
│   │   │   └── kraken-api.ts          // Domain operation → endpoint mapping
│   │   ├── auth/
│   │   │   └── signer.ts             // HMAC-SHA512 signature generation
│   │   ├── types/
│   │   │   └── index.ts              // Kraken API response types (raw shapes)
│   │   ├── models/
│   │   │   └── standardized.ts       // Re-exports from @cygnus-wealth/data-models
│   │   ├── mappers/
│   │   │   ├── balance-mapper.ts     // Kraken balances → Balance[]
│   │   │   ├── transaction-mapper.ts // Trades + Ledgers → Transaction[]
│   │   │   ├── position-mapper.ts    // Open positions → Balance[] with metadata
│   │   │   ├── staking-mapper.ts     // Staking data → StakedPosition[]
│   │   │   └── asset-mapper.ts       // Kraken assets → Asset, pair resolution
│   │   ├── services/
│   │   │   └── kraken-service.ts     // Facade: auth + API + mappers orchestration
│   │   ├── index.ts                  // Public API exports
│   │   └── __tests__/
│   │       ├── client.test.ts
│   │       ├── signer.test.ts
│   │       ├── kraken-api.test.ts
│   │       ├── kraken-service.test.ts
│   │       ├── balance-mapper.test.ts
│   │       ├── transaction-mapper.test.ts
│   │       ├── position-mapper.test.ts
│   │       ├── staking-mapper.test.ts
│   │       ├── asset-mapper.test.ts
│   │       └── e2e/
│   │           ├── fixtures/
│   │           │   └── kraken-responses.json
│   │           └── kraken-e2e.test.ts
│   ├── package.json
│   ├── tsconfig.json
│   ├── vitest.config.ts
│   ├── vitest.e2e.config.ts
│   ├── ARCHITECTURE.md
│   └── README.md
├── mayor/rig/                         // Replicated instance (same structure)
├── witness/
├── plugins/
└── config.json
```

### Layered Architecture

```
┌─────────────────────────────────────────────────┐
│                 KrakenService                     │
│        (Facade — public API surface)              │
│  authenticate() getPortfolio() getTransactions()  │
│  getBalances() getPositions() getStakingInfo()    │
└──────────────┬──────────────────┬────────────────┘
               │                  │
       ┌───────▼────────┐  ┌─────▼──────────┐
       │   KrakenAPI     │  │    Mappers      │
       │ (domain ops →   │  │ (Kraken →       │
       │  endpoint map)  │  │  DataModels)    │
       └───────┬─────────┘  └────────────────┘
               │
       ┌───────▼─────────┐
       │  KrakenClient   │
       │ (HTTP transport, │
       │  HMAC signing,   │
       │  rate limiting)  │
       └───────┬─────────┘
               │
       ┌───────▼─────────┐
       │   KrakenSigner  │
       │ (HMAC-SHA512,    │
       │  nonce mgmt)     │
       └─────────────────┘
```

### Layer Responsibilities

#### KrakenSigner (auth/signer.ts)

Pure cryptographic utility, no I/O:
- Accept `(apiSecret, uriPath, nonce, postData)` → return base64 `API-Sign`
- Use Web Crypto API (`crypto.subtle.importKey` + `crypto.subtle.sign`) for browser compatibility
- Fallback to Node.js `crypto.createHmac` for test environments
- Nonce generation with monotonic guard

#### KrakenClient (api/client.ts)

HTTP transport with auth and rate limiting:
- Wraps `fetch` (not axios — lighter for browser bundles)
- Injects `API-Key` and `API-Sign` headers on private endpoint calls
- Implements counter-based rate limiter matching Kraken's tier system
- Handles Kraken error responses (errors array in JSON body)
- Auto-pagination for history endpoints via `ofs` parameter
- Configurable timeout (default 30s), retry (default 3 attempts)

#### KrakenAPI (api/kraken-api.ts)

Maps domain operations to raw API calls:
- `getBalance()` → POST `/0/private/Balance`
- `getExtendedBalance()` → POST `/0/private/BalanceEx`
- `getTradeBalance()` → POST `/0/private/TradeBalance`
- `getOpenPositions()` → POST `/0/private/OpenPositions`
- `getTradesHistory(offset?)` → POST `/0/private/TradesHistory`
- `getLedgers(offset?, type?)` → POST `/0/private/Ledgers`
- `getClosedOrders(offset?)` → POST `/0/private/ClosedOrders`
- `getAssets()` → GET `/0/public/Assets`
- `getAssetPairs()` → GET `/0/public/AssetPairs`
- `getTicker(pairs)` → GET `/0/public/Ticker`
- `getStakingAssets()` → POST `/0/private/Staking/Assets`
- `getStakingTransactions()` → POST `/0/private/Staking/Transactions`
- `getStakingPending()` → POST `/0/private/Staking/Pending`
- `getTradeVolume()` → POST `/0/private/TradeVolume`

#### Mappers (mappers/*.ts)

Pure transformation functions (static methods, no state, no I/O):

- **AssetMapper**: Kraken asset name → standard symbol resolution using `/Assets` altname field. Caches the asset metadata map on first call. Resolves trading pairs from `/AssetPairs` into base/quote asset tuples.
- **BalanceMapper**: `KrakenBalance` + `KrakenExtendedBalance` → `Balance[]`. Filters zero-balance assets. Enriches with `Asset` metadata from AssetMapper.
- **TransactionMapper**: `KrakenTrade` / `KrakenLedgerEntry` → `Transaction`. Resolves pair names, maps buy/sell direction, calculates asset flow (assetsIn/assetsOut/fees).
- **PositionMapper**: `KrakenOpenPosition` → `Balance` with margin metadata. Calculates unrealized P&L from net field.
- **StakingMapper**: Staking assets + pending + transactions → `StakedPosition[]`. Maps APY, lock periods, reward history.

#### KrakenService (services/kraken-service.ts)

Facade orchestrating all layers:

- **`authenticate(credentials)`**: Stores API key/secret, makes a test call to `/0/private/Balance` to verify credentials. If 2FA required, emits `MFA_REQUIRED` error.
- **`getPortfolio()`**: Fetches balance + extended balance + trade balance + open positions + staking, assembles into `Portfolio` with three virtual accounts (spot, margin, staking).
- **`getBalances()`**: Fetches `/Balance` + `/BalanceEx`, maps via BalanceMapper.
- **`getTransactions(limit?)`**: Paginates through `/TradesHistory`, enriches with asset metadata, maps via TransactionMapper.
- **`getLedger(limit?, type?)`**: Paginates through `/Ledgers`, maps via TransactionMapper.
- **`getPositions()`**: Fetches `/OpenPositions`, maps via PositionMapper.
- **`getStakingPositions()`**: Fetches staking assets + pending, maps via StakingMapper.
- **`getFeeSchedule()`**: Fetches `/TradeVolume`, maps to `CexFeeSchedule`.
- **`isAuthenticated()`**: Returns whether valid credentials are loaded.

Error wrapping: All methods wrap errors in `StandardizedError` with `source: 'kraken'` and Kraken-specific error codes mapped to standard codes.

### Rate Limiting Implementation

Kraken's counter-based rate limiting requires a different approach than simple requests-per-second:

- Maintain an in-memory counter starting at 0
- Increment by 1 for standard calls, by 2 for history/ledger calls
- Decay the counter at the configured rate (default: Starter tier at -0.33/sec)
- Before each request, check if `counter + cost > maxCounter`; if so, wait until decay brings counter below threshold
- Tier configuration is user-settable (Starter: max 15, decay 0.33/s; Intermediate: max 20, decay 0.5/s; Pro: max 20, decay 1/s)

### Public API Exports

```
KrakenService              // Main service class (primary public API)
KrakenClient               // Low-level HTTP client (for advanced use)
KrakenAPI                  // API operation mapper
KrakenSigner               // HMAC signature utility
BalanceMapper              // Balance transformation
TransactionMapper          // Transaction transformation
PositionMapper             // Position transformation
StakingMapper              // Staking transformation
AssetMapper                // Asset name resolution
* from './types'           // All Kraken API response types
```

---

## 7. Error Handling

### Kraken Error Code Mapping

Kraken returns errors as an array in the JSON response body under the `error` key:

| Kraken Error | StandardizedError Code | Retriable | Action |
|-------------|----------------------|-----------|--------|
| `EAPI:Invalid key` | `AUTH_FAILED` | No | Prompt re-authentication |
| `EAPI:Invalid signature` | `AUTH_FAILED` | No | Check secret, prompt re-auth |
| `EAPI:Invalid nonce` | `NONCE_ERROR` | Yes | Reset nonce, retry |
| `EAPI:Rate limit exceeded` | `RATE_LIMITED` | Yes | Wait for counter decay, retry |
| `EService:Throttled` | `SERVICE_THROTTLED` | Yes | Wait until timestamp, retry |
| `EService:Unavailable` | `SERVICE_UNAVAILABLE` | Yes | Circuit breaker + retry |
| `EGeneral:Permission denied` | `PERMISSION_DENIED` | No | API key lacks required permission |
| `EQuery:Unknown asset pair` | `INVALID_ASSET` | No | Log and skip |

### Graceful Degradation

Following the Robinhood pattern:
- If staking endpoints fail, return portfolio without staking positions (log warning)
- If extended balance fails, fall back to basic balance endpoint
- If trade history pagination fails mid-way, return partial results with metadata indicating incompleteness
- If margin/position endpoints fail, return portfolio without margin account

---

## 8. Resilience Patterns

Applying the Integration Domain patterns from `patterns.md`:

### Circuit Breaker Configuration

```
failureThreshold: 5
successThreshold: 3
timeout: 30000ms
volumeThreshold: 10
errorFilter: Only count 5xx, network, and throttle errors (not auth errors)
```

### Retry Configuration

```
maxAttempts: 3
baseDelay: 1000ms
maxDelay: 30000ms
multiplier: 2
jitterFactor: 0.3
retryableErrors: [EAPI:Rate limit exceeded, EService:Throttled, EService:Unavailable, network errors]
nonRetryableErrors: [EAPI:Invalid key, EAPI:Invalid signature, EGeneral:Permission denied]
```

### Caching Strategy

| Data Type | TTL | Cache Layer |
|-----------|-----|-------------|
| Asset metadata (names, decimals) | 24 hours | L1 (memory) + L2 (IndexedDB) |
| Asset pairs | 24 hours | L1 + L2 |
| Spot balances | 60 seconds | L1 |
| Extended balances | 60 seconds | L1 |
| Trade balance (margin) | 60 seconds | L1 |
| Open positions | 60 seconds | L1 |
| Trade history | 5 minutes | L1 + L2 |
| Staking positions | 5 minutes | L1 |
| Fee schedule | 1 hour | L1 |

---

## 9. Integration with Portfolio Aggregation

### Contract Compliance

The `kraken-integration` package implements the same service contract pattern as `robinhood-integration`, making it pluggable into `portfolio-aggregation`:

- `getPortfolio()` returns a `Portfolio` containing `Account[]` with `Balance[]`
- `getTransactions()` returns `Transaction[]` using the unified asset flow model (assetsIn/assetsOut/fees)
- All types are from `@cygnus-wealth/data-models`

### Registration

Portfolio-aggregation's Integration Registry registers Kraken alongside Robinhood:

- **Integration ID**: `kraken`
- **Source**: `IntegrationSource.KRAKEN`
- **Capabilities**: `['balance', 'transactions', 'positions', 'staking']`
- **Credential Type**: API key + secret (not OAuth)
- **Refresh Strategy**: Poll-based (no WebSocket — CEX APIs are HTTP-only for account data)

### Data Priority

From existing architecture: "Prefer on-chain data over CEX data." If the same asset appears in both an on-chain wallet and Kraken, the on-chain balance is authoritative. Kraken balances are additive to the portfolio, not conflicting.

---

## 10. Testing Strategy

### Test Distribution

Following the Integration Domain testing pyramid:

- **Unit Tests (80%)**: Signer, mappers, client (mocked fetch), service (mocked API)
- **Integration Tests (15%)**: Full stack with mocked HTTP responses (recorded fixtures)
- **Contract Tests (4%)**: Verify all mapper outputs conform to DataModels interfaces
- **E2E Tests (1%)**: Critical path — authenticate + fetch portfolio + verify data shape

### Critical Test Scenarios

1. **HMAC signature correctness**: Verify against known test vectors from Kraken docs
2. **Nonce monotonicity**: Rapid sequential requests produce strictly increasing nonces
3. **Asset name resolution**: `XXBT` → `BTC`, `ZUSD` → `USD`, `SOL` → `SOL`
4. **Rate limit counter**: Counter increments, decays, and blocks correctly
5. **Pagination completeness**: 150 trades across 3 pages fetched completely
6. **Zero-balance filtering**: Assets with 0 balance excluded from portfolio
7. **Margin P&L calculation**: Unrealized gains calculated from position net field
8. **Staking reward accumulation**: Multiple reward ledger entries summed correctly
9. **Error mapping**: All Kraken error codes map to correct StandardizedError codes
10. **Graceful degradation**: Partial endpoint failures return partial portfolio

---

## 11. Implementation Sequence

1. **KrakenSigner** — HMAC-SHA512 signing with Web Crypto API + unit tests with known test vectors
2. **KrakenClient** — HTTP transport with signing integration, counter-based rate limiter
3. **Kraken raw types** — TypeScript interfaces for all API response shapes
4. **AssetMapper** — Asset name resolution and pair splitting (foundational for all other mappers)
5. **KrakenAPI** — Endpoint mapping layer
6. **BalanceMapper + TransactionMapper + PositionMapper + StakingMapper** — Data transformations
7. **KrakenService** — Facade orchestration with error wrapping and graceful degradation
8. **E2E tests with recorded fixtures**
9. **Portfolio-aggregation integration** — Register Kraken as a new integration source
10. **CygnusWealthApp UI** — Add Kraken connection flow (API key input, optional OTP)

---

## Related Documentation

- **[Integration Domain README](../domains/integration/README.md)** — Domain overview and strategic guidance
- **[Integration Patterns](../domains/integration/patterns.md)** — Anti-corruption layer, circuit breaker, retry, caching patterns
- **[Resilience & Performance](../domains/integration/resilience-performance.md)** — Connection management and fallback strategies
- **[Testing & Security](../domains/integration/testing-security.md)** — Testing pyramid and security requirements
- **[Robinhood Integration](../domains/integration/bounded-contexts/robinhood-integration.md)** — Reference CEX integration pattern
- **[Data Models](../domains/contract/bounded-contexts/data-models.md)** — Unified type definitions
- **[Repository Organization](../repository-organization.md)** — GitHub repo setup and topic tagging
- **[Domain Contracts](../contracts.md)** — Inter-domain communication contracts
