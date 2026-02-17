# en-25w5: RPC Provider Fallback Chain Implementation Directive

**Status**: IMPLEMENTATION DIRECTIVE
**Date**: 2026-02-17
**Bead**: en-25w5
**Scope**: RPC provider initialization, fallback chain, circuit breakers, API key management, config propagation from app to integration libs

---

## 1. Problem Statement

### What Was Written

`rpc-strategy.md` defines a comprehensive multi-provider fallback architecture:
- Per-chain provider priority (Primary → Secondary → Tertiary → Emergency → Cache → Offline)
- Circuit breaker thresholds (5 failures / 60s → OPEN, 30s timeout, 3 successes to close)
- Timeout hierarchy (5s managed, 10s public, 30s total operation)
- Health monitoring with P50/P95/P99 latency tracking and dynamic promotion/demotion
- Provider selection for 9 chains (8 EVM + Solana) across 6 providers (Alchemy, dRPC, Helius, QuickNode, Infura, Ankr)

`patterns.md` defines circuit breaker state machine, retry with exponential backoff, token bucket rate limiting, connection pooling, and request coalescing patterns.

`resilience-performance.md` defines the FallbackChain class, BulkheadManager, TimeoutManager, HealthMonitor, and WeightedLoadBalancer patterns.

### What Was Never Implemented

None of it. The integration libraries have no:
- `RpcProviderConfig` type in `data-models`
- Provider initialization from environment variables
- Fallback chain orchestrator
- Circuit breaker per provider per chain
- Rate limiter per provider
- Health check infrastructure
- Configuration propagation path from `cygnus-wealth-app` to integration libs

### The Symptom

The app hits public RPCs directly (e.g., `api.mainnet-beta.solana.com`), which returns 403 when rate-limited. There is no fallback — a single provider failure means total data loss for that chain.

### Root Cause

The architecture documents define WHAT to build but never specified the implementation seam: how config enters the system, flows through layers, and initializes provider infrastructure. The integration bounded context specs (evm-integration.md, sol-integration.md) mention "configurable RPC endpoints" and "automatic failover to backup endpoints" but provide no specification for how configuration is structured, where it lives, or how it propagates from the app shell to the integration libraries.

---

## 2. Implementation Specification

### 2.1 Standard RpcProviderConfig Type in data-models

The Contract Domain (`data-models`) must define the RPC provider configuration types that all integration libraries consume. These are shared contracts — any integration lib that accepts RPC configuration uses these types.

#### Types to Add

**`RpcProviderRole`** — the priority tier of a provider in the fallback chain:
- `primary` — first attempt, expected best performance (managed provider with API key)
- `secondary` — first fallback (different managed provider)
- `tertiary` — second fallback (another managed or public provider)
- `emergency` — last-resort public RPCs (no auth, best-effort)

**`RpcProviderType`** — classification of the provider:
- `managed` — requires API key, has SLA/rate limits (Alchemy, dRPC, Helius, QuickNode, Infura)
- `public` — authenticated but free-tier public endpoints (Ankr public RPCs)
- `community` — fully open, no auth, heavily rate-limited (llamarpc, 1rpc.io, chain-native RPCs)

**`RpcEndpointConfig`** — a single RPC endpoint:
- `url: string` — the HTTP(S) endpoint URL. For managed providers, this includes the API key in the URL path (e.g., `https://eth-mainnet.g.alchemy.com/v2/${apiKey}`) or as a query parameter, depending on the provider
- `wsUrl?: string` — optional WebSocket endpoint URL for real-time subscriptions
- `provider: string` — provider identifier (e.g., `'alchemy'`, `'drpc'`, `'helius'`, `'infura'`, `'ankr'`, `'quicknode'`, `'public'`)
- `role: RpcProviderRole` — the tier in the fallback chain
- `type: RpcProviderType` — managed vs public vs community
- `rateLimitRps: number` — maximum requests per second for this endpoint
- `timeoutMs: number` — per-request timeout in milliseconds
- `weight?: number` — relative weight for weighted load balancing within the same role (default: 1)

**`ChainRpcConfig`** — the complete RPC configuration for a single chain:
- `chainId: string` — chain identifier (e.g., `'ethereum'`, `'polygon'`, `'solana'`)
- `chainName: string` — human-readable name
- `endpoints: RpcEndpointConfig[]` — ordered by role (primary first, emergency last)
- `totalOperationTimeoutMs: number` — maximum time across all fallback attempts (default: 30000)
- `cacheStaleAcceptanceMs: Record<string, number>` — how long stale cached data is acceptable per data type (e.g., `{ balance: 300000, metadata: 3600000, transaction: 300000, gasPrice: 10000 }`)

**`RpcProviderConfig`** — the top-level configuration object consumed by integration libraries:
- `chains: Record<string, ChainRpcConfig>` — keyed by chainId
- `circuitBreaker: CircuitBreakerConfig` — shared circuit breaker defaults (can be overridden per endpoint)
- `retry: RetryConfig` — shared retry defaults
- `healthCheck: HealthCheckConfig` — health monitoring configuration

**`CircuitBreakerConfig`**:
- `failureThreshold: number` — failures before opening (default: 5)
- `failureWindowMs: number` — rolling window for counting failures (default: 60000)
- `successThreshold: number` — successes in HALF_OPEN to close (default: 3)
- `openDurationMs: number` — time in OPEN before transitioning to HALF_OPEN (default: 30000)
- `volumeThreshold: number` — minimum requests before stats are meaningful (default: 10)

**`RetryConfig`**:
- `maxAttempts: number` — max retries per request within a single provider (default: 2)
- `baseDelayMs: number` — initial backoff delay (default: 1000)
- `maxDelayMs: number` — maximum backoff delay (default: 10000)
- `multiplier: number` — exponential multiplier (default: 2)
- `jitterFactor: number` — jitter as fraction of base delay (default: 0.3)

**`HealthCheckConfig`**:
- `intervalMs: number` — how often to ping providers (default: 60000)
- `timeoutMs: number` — health check timeout (default: 5000)
- `enabled: boolean` — whether background health checks are active (default: true)

#### Location in data-models

These types go in a new section of the data-models bounded context under **Network Configuration**. The data-models.md already lists "Network Configuration: RPC endpoints and chain parameters" as a planned model category — this fulfills that plan.

File: `data-models/src/network/rpc-config.ts` (when data-models is implemented as a package)

In the architecture docs, add these type definitions to data-models.md under a new "### RPC Provider Configuration" subsection of "### Chain and Network Models".

#### What Does NOT Go in data-models

- Actual API key values (those are runtime secrets, not types)
- Provider-specific URL construction logic (that's an adapter concern in each integration lib)
- Circuit breaker state or health monitor state (those are runtime, not configuration)
- The FallbackChain class itself (that's an implementation pattern in each integration lib)

---

### 2.2 Where API Keys Live

#### Environment Variable Naming Convention

All RPC API keys are stored as environment variables with a strict naming convention:

```
CYGNUS_RPC_{PROVIDER}_{OPTIONAL_CHAIN}_API_KEY
```

**Provider-scoped keys** (one key works across all chains for that provider):
- `CYGNUS_RPC_ALCHEMY_API_KEY` — Alchemy API key (works for all EVM chains + Solana)
- `CYGNUS_RPC_DRPC_API_KEY` — dRPC API key
- `CYGNUS_RPC_INFURA_API_KEY` — Infura project ID
- `CYGNUS_RPC_QUICKNODE_API_KEY` — QuickNode API key
- `CYGNUS_RPC_HELIUS_API_KEY` — Helius API key (Solana only)

**Chain-scoped overrides** (when a chain uses a different key for the same provider):
- `CYGNUS_RPC_ALCHEMY_POLYGON_API_KEY` — override for Polygon-specific Alchemy key
- These are optional — if absent, the provider-scoped key is used

#### .env File Structure

For local development, keys are stored in `.env` at the repository root of `cygnus-wealth-app`:

```
# RPC Provider API Keys
CYGNUS_RPC_ALCHEMY_API_KEY=your-alchemy-key
CYGNUS_RPC_DRPC_API_KEY=your-drpc-key
CYGNUS_RPC_HELIUS_API_KEY=your-helius-key
CYGNUS_RPC_INFURA_API_KEY=your-infura-project-id
CYGNUS_RPC_QUICKNODE_API_KEY=your-quicknode-key
```

The `.env` file is gitignored. A `.env.example` file with empty values and comments documents the required variables.

#### How Keys Are Injected Into URLs

Each provider has a different URL pattern. The URL construction is the responsibility of a **provider URL builder** utility, NOT the configuration types:

| Provider | URL Pattern |
|----------|-------------|
| Alchemy | `https://{chain-slug}.g.alchemy.com/v2/{apiKey}` |
| dRPC | `https://lb.drpc.org/ogrpc?network={chain-slug}&dkey={apiKey}` |
| Helius | `https://mainnet.helius-rpc.com/?api-key={apiKey}` |
| Infura | `https://{chain-slug}.infura.io/v3/{apiKey}` |
| QuickNode | Custom subdomain per project (stored as full URL in env) |
| Ankr (public) | `https://rpc.ankr.com/{chain-slug}` (no key) |
| Community | Full URLs stored directly (no key injection) |

The provider URL builder reads environment variables, applies the provider's URL template, and produces `RpcEndpointConfig` objects with fully-resolved URLs. This builder runs once at app initialization.

#### Environment Separation

Per rpc-strategy.md Section "API Key Management":
- Separate API keys per environment (dev, CI, staging, production)
- Different `.env` files or CI secret stores per environment
- The configuration type system is the same — only the key values differ

---

### 2.3 How CygnusWealthApp Passes Provider Config to Integration Libs

#### The Configuration Flow

```
.env file (API keys)
    │
    ▼
cygnus-wealth-app initialization
    │
    ├─► Provider URL Builder
    │     Reads env vars
    │     Builds full endpoint URLs per provider per chain
    │     Produces: Record<string, ChainRpcConfig>
    │
    ├─► RpcProviderConfig assembly
    │     Combines ChainRpcConfig map with default circuit breaker/retry/health configs
    │     Produces: RpcProviderConfig
    │
    ▼
Dependency injection into integration libraries
    │
    ├─► evm-integration.initialize(rpcConfig: RpcProviderConfig)
    │     Extracts EVM chain configs: ethereum, polygon, arbitrum, optimism, bnb, avalanche, base, fantom
    │     Creates per-chain RpcFallbackChain instances
    │
    └─► sol-integration.initialize(rpcConfig: RpcProviderConfig)
          Extracts Solana chain config
          Creates Solana-specific RpcFallbackChain with Helius as primary
```

#### The App-Level Configuration Bootstrap

`cygnus-wealth-app` is the composition root. It is the ONLY place that reads environment variables and constructs the provider configuration. Integration libraries NEVER read env vars directly — they receive a fully-formed `RpcProviderConfig` as a constructor/initialization parameter.

This follows the dependency inversion principle already established in contracts.md: dependencies flow inward, and the experience domain (app) orchestrates initialization.

**Bootstrap sequence in cygnus-wealth-app**:

1. App starts, reads `.env` / environment variables
2. Calls `buildRpcProviderConfig(env)` — a utility function (lives in cygnus-wealth-app or a shared config package) that:
   - Reads all `CYGNUS_RPC_*` env vars
   - For each chain defined in the default chain registry, builds the endpoint list using the provider URL builder
   - Falls back to public/community endpoints if API keys are absent (development mode)
   - Assembles the complete `RpcProviderConfig`
3. Passes the `RpcProviderConfig` to each integration library's initialization function
4. Integration libraries create their internal infrastructure (fallback chains, circuit breakers, health monitors) from the config

#### Default Configuration (When API Keys Are Missing)

If no API keys are provided (bare development environment), the config builder produces a degraded configuration using only public/community endpoints. This is functional but heavily rate-limited — exactly what happens today, except now it's explicit rather than accidental.

Each chain always has at least one emergency community endpoint defined as a static default. These are the public RPCs listed in rpc-strategy.md:
- Ethereum: `https://eth.llamarpc.com`, `https://1rpc.io/eth`
- Polygon: `https://polygon-rpc.com`, `https://1rpc.io/matic`
- Solana: `https://api.mainnet-beta.solana.com` (with explicit warning about rate limits)
- etc.

This means the app never fails at initialization due to missing keys — it degrades to community RPCs with appropriate logging.

#### React Context Integration

In the React 19 app, the RPC configuration is provided via a context provider near the app root:

```
<RpcConfigProvider config={rpcProviderConfig}>
  <PortfolioProvider>
    <App />
  </PortfolioProvider>
</RpcConfigProvider>
```

Integration libraries access the config via their initialization interface, NOT by reaching into React context themselves. The app shell calls the integration library factory with the config object. Integration libraries are framework-agnostic — they accept config as plain TypeScript objects.

---

### 2.4 How Each Integration Lib Initializes RPC Providers from Config

#### evm-integration Initialization

evm-integration receives the full `RpcProviderConfig` and extracts only EVM chain configurations. It creates a per-chain `RpcFallbackChain` instance.

**Initialization contract**:

`createEvmIntegration(config: RpcProviderConfig): EvmIntegrationService`

This factory function:
1. Filters `config.chains` to recognized EVM chain IDs: `ethereum`, `polygon`, `arbitrum`, `optimism`, `bnb`, `avalanche`, `base`, `fantom`
2. For each chain, creates:
   - One `CircuitBreaker` instance per endpoint (keyed by `{chainId}:{provider}`)
   - One `TokenBucket` rate limiter per endpoint (initialized from `rateLimitRps`)
   - One `RpcFallbackChain` that wraps the ordered endpoint list with circuit breakers
   - One entry in the `HealthMonitor` for periodic pings
3. Creates the shared `HierarchicalCache` (L1 memory, L2 IndexedDB)
4. Returns the service facade with `getBalances()`, `getTransactions()`, `getTokens()`, etc.

**Per-chain RPC call path**:

```
BalanceService.getBalance(address, chainId)
    │
    ▼
RpcFallbackChain.execute(chainId, rpcMethod, params)
    │
    ├─► Check L1/L2 cache first
    │     Hit? → return cached, schedule background refresh if stale
    │
    ├─► Try endpoint[0] (primary)
    │     CircuitBreaker check → if OPEN, skip
    │     TokenBucket.acquire() → if no tokens, skip
    │     Execute RPC call with endpoint-specific timeout
    │     Success? → update cache, record latency, return
    │     Failure? → record failure in circuit breaker, continue
    │
    ├─► Try endpoint[1] (secondary)
    │     Same flow
    │
    ├─► Try endpoint[2] (tertiary)
    │     Same flow
    │
    ├─► Try endpoint[3..n] (emergency)
    │     Timeout extended to 10s for public RPCs
    │     Same flow
    │
    ├─► Check cache for stale-but-acceptable data
    │     Within cacheStaleAcceptanceMs? → return with staleness metadata
    │
    └─► Throw NoDataAvailableError with chain context
```

**ethers.js provider management**: For EVM chains, each endpoint maps to an `ethers.JsonRpcProvider` (or `ethers.WebSocketProvider` for WS endpoints). The `RpcFallbackChain` does NOT create a single `ethers.FallbackProvider` — instead it manages failover itself, because `ethers.FallbackProvider` doesn't support circuit breakers, rate limiting, or stale cache fallback. Each provider instance is created lazily on first use and cached for connection reuse.

#### sol-integration Initialization

sol-integration follows the same pattern but with Solana-specific differences.

**Initialization contract**:

`createSolIntegration(config: RpcProviderConfig): SolIntegrationService`

This factory function:
1. Extracts `config.chains['solana']`
2. Creates per-endpoint infrastructure (circuit breaker, rate limiter) — same as EVM
3. Creates the `RpcFallbackChain` for Solana
4. Returns the service facade

**Solana-specific differences**:
- Uses `@solana/web3.js` `Connection` class instead of ethers
- Helius as primary provider enables DAS API methods (getAssetsByOwner, etc.) that other providers don't support — the fallback chain must be aware that DAS-specific calls can only fall back to other DAS-supporting providers or return cached data
- The health check for Solana uses `getHealth()` RPC method (Solana-specific) rather than a generic block number check
- Staked connection support (Helius paid plans) is a per-endpoint capability, tracked in endpoint metadata

**DAS API fallback handling**: When the fallback chain encounters a DAS-specific method (e.g., `getAssetsByOwner`), it restricts fallback to endpoints that support DAS. If no DAS-capable endpoint is available, it falls back to the legacy method equivalent (e.g., `getTokenAccountsByOwner` + metadata fetch) or cached data. This is a method-level concern within the Solana fallback chain, not a configuration concern.

---

### 2.5 Circuit Breaker + Fallback Chain Implementation Pattern

#### Circuit Breaker — Per Endpoint, Per Chain

Each `(chainId, provider)` pair gets its own circuit breaker instance. This means Alchemy-Ethereum and Alchemy-Polygon have independent circuit breakers — an Alchemy Ethereum outage doesn't affect Alchemy Polygon.

**State machine**:

```
CLOSED (normal operation)
    │
    │ failures >= failureThreshold within failureWindowMs
    ▼
OPEN (fast-fail, skip this endpoint)
    │
    │ openDurationMs elapsed
    ▼
HALF_OPEN (probe with single request)
    │
    ├─► successThreshold consecutive successes → CLOSED
    └─► any failure → OPEN (reset timer)
```

**Failure counting**: Uses a rolling window (ring buffer of timestamps). Only failures within the last `failureWindowMs` count toward the threshold. This prevents ancient failures from keeping a circuit open.

**What counts as a failure**:
- HTTP 5xx responses
- Connection timeouts
- Connection refused / network errors
- HTTP 429 (rate limited) — counts as failure AND triggers immediate backoff
- HTTP 403 (forbidden) — counts as failure, likely indicates revoked/invalid API key

**What does NOT count as a failure**:
- HTTP 4xx other than 429/403 (client errors — likely bad request, not provider issue)
- Successful responses with application-level errors (e.g., "execution reverted" from an EVM call — this is valid data, not a provider failure)

#### Fallback Chain — Per Chain

Each chain has one `RpcFallbackChain` instance containing the ordered endpoint list. The chain is the unit of fallback — when calling `getBalance` for Ethereum, the Ethereum fallback chain tries Alchemy → dRPC → Ankr → public RPCs → cache.

**Key behaviors**:

1. **Skip OPEN circuit breakers**: If the primary endpoint's circuit breaker is OPEN, immediately try the secondary. No wasted time.

2. **Rate limit check before attempt**: Before sending a request to an endpoint, check the token bucket. If the bucket is empty, skip to the next endpoint rather than waiting. This prevents queuing behind a rate-limited provider when alternatives are available.

3. **Total operation timeout**: The fallback chain enforces the `totalOperationTimeoutMs` across ALL attempts. If the total timeout expires mid-chain, stop trying and fall through to cache.

4. **Cache on success**: Every successful response updates the cache (L1 + L2). This ensures cache freshness for fallback scenarios.

5. **Stale cache acceptance**: After all endpoints fail, the cache is checked with chain-specific staleness thresholds from `cacheStaleAcceptanceMs`. Stale data is returned with metadata indicating staleness, so the UI can display "last updated X minutes ago."

6. **Offline mode**: If all endpoints fail AND no acceptable cached data exists, the chain returns a `NoDataAvailableError` with the chain ID, last attempted endpoint, and last error. The UI translates this to an offline indicator for that chain.

#### Health Monitoring — Background Probing

When `healthCheck.enabled` is true, a background interval pings all endpoints:

**Health check method**:
- EVM chains: `eth_blockNumber` — lightweight, returns current block number
- Solana: `getHealth` — returns "ok" if the node is healthy

**What health checks enable**:
- **Faster recovery**: When a circuit breaker is OPEN, health checks detect recovery before the next user request. The circuit transitions to HALF_OPEN immediately on a successful health check rather than waiting for the full `openDurationMs`.
- **Dynamic promotion**: If the secondary provider consistently outperforms the primary (lower latency in health checks), the fallback chain can swap their order. This is an optional optimization — not required for MVP but the infrastructure should support it.
- **Proactive alerting**: If all managed providers for a chain are unhealthy, log a warning. The app can display a degraded-mode indicator.

**Health check implementation constraint**: Health checks must NOT consume significant rate limit budget. One lightweight call per provider per `intervalMs` (default: 60s) is ~1 request/minute per endpoint. For 6 providers across 9 chains, that's ~54 requests/minute total — negligible.

#### Retry Within a Single Endpoint

Before moving to the next endpoint in the fallback chain, a single endpoint gets `maxAttempts` tries with exponential backoff:

```
Attempt 1: immediate
Attempt 2: baseDelayMs * 2^1 + jitter = ~2000ms + jitter
Attempt 3: baseDelayMs * 2^2 + jitter = ~4000ms + jitter (capped at maxDelayMs)
```

**When NOT to retry**:
- HTTP 403 (auth failure — retrying won't help)
- HTTP 429 with Retry-After header > remaining operation timeout
- Circuit breaker tripped during retry sequence
- Total operation timeout approaching

**Default: maxAttempts = 2** (1 initial + 1 retry). This keeps fallback fast — better to try the next provider than burn time retrying a failing one.

---

## 3. Default Chain Configurations

These are the default `ChainRpcConfig` entries, derived directly from rpc-strategy.md. The config builder produces these when the corresponding API keys are present in environment variables.

### Ethereum Mainnet

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | ankr | public | 30 | 5000ms |
| emergency | public (llamarpc) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Polygon

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | ankr | public | 30 | 5000ms |
| emergency | public (polygon-rpc.com) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Arbitrum One

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | infura | managed | 15 | 5000ms |
| emergency | public (arb1.arbitrum.io) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Optimism

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | infura | managed | 15 | 5000ms |
| emergency | public (mainnet.optimism.io) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### BNB Smart Chain

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | ankr | public | 30 | 5000ms |
| emergency | public (bsc-dataseed.binance.org) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Avalanche C-Chain

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | infura | managed | 15 | 5000ms |
| emergency | public (api.avax.network) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Base

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | alchemy | managed | 25 | 5000ms |
| secondary | drpc | managed | 100 | 5000ms |
| tertiary | infura | managed | 15 | 5000ms |
| emergency | public (mainnet.base.org) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |

### Fantom Opera

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | drpc | managed | 100 | 5000ms |
| secondary | ankr | public | 30 | 5000ms |
| tertiary | public (rpcapi.fantom.network) | community | 10 | 10000ms |
| emergency | public (1rpc) | community | 10 | 10000ms |
| emergency | public (fantom-mainnet.public.blastapi.io) | community | 10 | 10000ms |

Note: Fantom has no Alchemy support (deprecated). dRPC is primary.

### Solana

| Role | Provider | Type | RPS Limit | Timeout |
|------|----------|------|-----------|---------|
| primary | helius | managed | 10 | 5000ms |
| secondary | alchemy | managed | 25 | 5000ms |
| tertiary | drpc | managed | 100 | 5000ms |
| quaternary | infura | managed | 15 | 5000ms |
| emergency | public (api.mainnet-beta.solana.com) | community | 5 | 10000ms |

Note: Helius free tier is 10 RPS. DAS API methods only available on Helius (primary) — DAS calls cannot fall back to other providers, only to cache or legacy method equivalents.

---

## 4. Implementation Sequencing

### Phase 1: data-models — RPC Configuration Types

**What**: Add the `RpcProviderConfig` type family to the Contract Domain.
**Where**: `data-models` package, network configuration section.
**Dependencies**: None — this is the foundation.
**Produces**: Importable TypeScript types for all downstream consumers.
**Verification**: Type-check only. No runtime behavior.

### Phase 2: cygnus-wealth-app — Config Bootstrap + Provider URL Builder

**What**: Implement `buildRpcProviderConfig(env)` that reads `CYGNUS_RPC_*` env vars, constructs endpoint URLs per provider URL templates, and assembles the complete `RpcProviderConfig`.
**Where**: `cygnus-wealth-app`, likely in a `src/config/` or `src/infrastructure/` directory.
**Dependencies**: Phase 1 (types).
**Produces**: A function that returns `RpcProviderConfig` from environment variables.
**Includes**: `.env.example` with all supported variables documented.
**Verification**: Unit tests that mock env vars and assert correct config assembly. Tests for missing keys (degraded mode), partial keys, and full keys.

### Phase 3: Integration Lib Core — RpcFallbackChain + CircuitBreaker + RateLimiter

**What**: Implement the shared infrastructure classes that both `evm-integration` and `sol-integration` use.
**Where**: This could live in a shared `@cygnus-wealth/rpc-infrastructure` package, OR be duplicated in each integration lib. Recommendation: shared package, since the circuit breaker, rate limiter, and fallback chain logic is identical between EVM and Solana — only the health check method differs.
**Key classes**:
- `CircuitBreaker` — state machine per (chainId, provider)
- `TokenBucketRateLimiter` — per-endpoint rate limiting
- `RpcFallbackChain` — ordered endpoint list with circuit breaker + rate limiter integration, cache fallback, total timeout enforcement
- `HealthMonitor` — background pinging with configurable health check method
- `ProviderMetrics` — latency tracking (P50/P95/P99), error rate calculation
**Dependencies**: Phase 1 (types).
**Verification**: Extensive unit tests. Circuit breaker state transitions. Rate limiter token refill. Fallback chain endpoint ordering. Timeout enforcement. Cache fallback behavior.

### Phase 4: evm-integration — Provider Initialization + Fallback Integration

**What**: Wire `createEvmIntegration(config)` factory. Create per-chain `ethers.JsonRpcProvider` instances lazily. Wrap all RPC calls through `RpcFallbackChain`.
**Where**: `evm-integration` package.
**Dependencies**: Phase 1 (types), Phase 3 (infrastructure).
**Key changes**:
- Factory function accepts `RpcProviderConfig`, extracts EVM chains
- BalanceService, TransactionService, TrackingService route all RPC calls through the per-chain `RpcFallbackChain`
- ethers providers created lazily on first call per endpoint
- Remove any hardcoded RPC URLs
**Verification**: Integration tests with mocked RPC responses simulating primary failure → secondary success, all-fail → cache, circuit breaker opening.

### Phase 5: sol-integration — Provider Initialization + Fallback Integration

**What**: Wire `createSolIntegration(config)` factory. Create per-endpoint `@solana/web3.js Connection` instances. Wrap all RPC calls through `RpcFallbackChain`.
**Where**: `sol-integration` package.
**Dependencies**: Phase 1 (types), Phase 3 (infrastructure).
**Key changes**:
- Factory function accepts `RpcProviderConfig`, extracts Solana config
- DAS-aware fallback: methods that require DAS API (getAssetsByOwner) have a restricted fallback path
- Helius-specific health check using `getHealth()`
- Remove any hardcoded reference to `api.mainnet-beta.solana.com` as a primary endpoint
**Verification**: Same as Phase 4 but with Solana-specific scenarios (DAS fallback, staked connection awareness).

### Phase 6: cygnus-wealth-app — Wiring

**What**: Connect the bootstrap config to the integration library factories. Pass `RpcProviderConfig` through the app initialization sequence.
**Where**: `cygnus-wealth-app` app shell / composition root.
**Dependencies**: Phases 2, 4, 5.
**Key changes**:
- Call `buildRpcProviderConfig(import.meta.env)` at app start
- Pass config to `createEvmIntegration(config)` and `createSolIntegration(config)`
- Expose integration services to the app via React context or dependency injection
**Verification**: E2E smoke test — app starts with valid API keys and fetches a balance from at least one chain.

### Dependency Order

```
Phase 1 (data-models types)
    │
    ├──► Phase 2 (app config bootstrap)
    │
    └──► Phase 3 (RPC infrastructure)
              │
              ├──► Phase 4 (evm-integration)
              │
              └──► Phase 5 (sol-integration)
                        │
                        ▼
                  Phase 6 (app wiring)
```

Phases 2 and 3 can proceed in parallel after Phase 1.
Phases 4 and 5 can proceed in parallel after Phase 3.
Phase 6 requires all prior phases.

---

## 5. Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Shared `rpc-infrastructure` package adds a dependency. If either integration lib needs divergent behavior, the shared package becomes a constraint. | Start with shared package for the 90% common case. If divergence is needed, fork the specific class into the integration lib. The shared package provides defaults, not mandates. |
| Environment variable proliferation — 5+ keys to manage. | `.env.example` documents every variable. Config builder logs which providers are active vs missing at startup. Degraded mode works without any keys. |
| Lazy provider creation means the first request to each endpoint is slow (connection establishment). | Health monitor warms connections during its first check cycle (60s after startup). For critical chains (Ethereum, Solana), eagerly create the primary provider at initialization. |
| Circuit breaker thresholds may need tuning per chain (Fantom public RPCs are flakier than Ethereum managed RPCs). | Default thresholds from `CircuitBreakerConfig` are global. Per-endpoint overrides can be added to `RpcEndpointConfig` as `circuitBreakerOverrides?: Partial<CircuitBreakerConfig>`. Not needed for MVP — tune based on production metrics. |
| Total operation timeout (30s) may be too short when all managed providers are down and only public RPCs remain with 10s timeouts. | 30s allows for: 5s primary + 5s secondary + 10s tertiary + 10s emergency = 30s. If this proves too tight, the per-chain `totalOperationTimeoutMs` is configurable. Stale cache acceptance extends the effective availability beyond the timeout. |
| `api.mainnet-beta.solana.com` returning 403 even as emergency endpoint. | This is the known symptom. The fallback chain explicitly deprioritizes it as the last resort. With Helius, Alchemy, dRPC, and Infura all ahead of it, the public endpoint should rarely be reached. When it IS reached and returns 403, the circuit breaker opens after 5 failures, and the cache fallback activates. |
| Integration libs currently may have hardcoded RPC URLs scattered through the codebase. | Phase 4/5 explicitly includes "remove any hardcoded RPC URLs" as a key change. The factory pattern ensures all RPC access flows through the configured fallback chain. |

---

## 6. Relationship to Existing Architecture Documents

This directive operationalizes specifications from multiple existing documents:

| Document | What It Defined | What This Directive Adds |
|----------|----------------|--------------------------|
| rpc-strategy.md | Provider selection per chain, fallback tiers, rate limits | How config is structured, where keys live, how the app passes config to libs |
| patterns.md | Circuit breaker interface, retry config, token bucket, connection pool | How these patterns are instantiated per endpoint per chain, lifecycle management |
| resilience-performance.md | FallbackChain class, BulkheadManager, HealthMonitor | The concrete initialization path and wiring between these components |
| evm-integration.md | "Configurable RPC endpoints", "automatic failover" | The factory function contract and per-chain RPC call path |
| sol-integration.md | "Primary RPC endpoint with automatic fallback" | Solana-specific fallback behavior (DAS awareness, getHealth) |
| data-models.md | "Network Configuration: RPC endpoints and chain parameters" | The actual type definitions that fulfill this planned model category |
| contracts.md | App → Integration dependency direction | How config propagates from app composition root to integration libraries |

No existing architectural decision is changed. This directive fills the implementation gap between the documented patterns and actual running code.

---

## Summary

| Question | Answer |
|----------|--------|
| Where do RPC config types live? | `data-models` — `RpcProviderConfig` type family under Network Configuration |
| Where do API keys live? | Environment variables (`CYGNUS_RPC_{PROVIDER}_API_KEY`), `.env` file for local dev |
| Who reads env vars? | `cygnus-wealth-app` only — via `buildRpcProviderConfig(env)` |
| How do integration libs get config? | Factory function parameter: `createEvmIntegration(config: RpcProviderConfig)` |
| How does fallback work? | `RpcFallbackChain` per chain: primary → secondary → tertiary → emergency → stale cache → offline |
| How does circuit breaking work? | One `CircuitBreaker` per (chainId, provider). 5 failures / 60s → OPEN. 30s → HALF_OPEN. 3 successes → CLOSED. |
| How does rate limiting work? | One `TokenBucket` per endpoint, initialized from `rateLimitRps`. Empty bucket = skip to next endpoint. |
| What happens with no API keys? | Degraded mode — community endpoints only. Functional but heavily rate-limited. Explicit logging. |
| What about the Solana 403 problem? | Helius/Alchemy/dRPC/Infura all precede the public endpoint. Public endpoint circuit breaks after 5 failures. Cache fallback prevents total data loss. |
