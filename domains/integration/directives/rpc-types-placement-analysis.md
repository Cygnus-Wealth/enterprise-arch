# en-1mg1: Should RPC Provider Types Live in data-models?

**Status**: ARCHITECTURAL ANALYSIS
**Date**: 2026-02-20
**Bead**: en-1mg1
**Scope**: Evaluate placement of RPC infrastructure types (`RpcProviderConfig`, `CircuitBreakerConfig`, `RetryConfig`, etc.) in the Contract Domain shared kernel (`data-models`)
**Evaluates**: en-25w5 Section 2.1, en-tfkn Section 8

---

## Verdict

**No. The original placement was a mistake.** The en-25w5 directive conflated two different categories of shared types: domain model types that belong in the shared kernel, and infrastructure interface types that belong at the integration boundary. Strict DDD analysis says the RPC provider configuration types should not live in `data-models`.

This is not a catastrophic error — the types are pure (no runtime behavior) and the system hasn't been built yet. But if left uncorrected before implementation begins, it will embed an infrastructure coupling into the most stable, most-depended-upon layer of the architecture.

---

## 1. What data-models Is

The `data-models` bounded context is the Contract Domain's shared kernel. Its own specification defines its scope:

> "Unified data structures ensuring consistency and clear communication between all domains"

Its core model categories are:
- **Portfolio Models**: Portfolio, Asset, Balance, Value
- **Transaction Models**: Transaction, types, status, metadata
- **Market Data Models**: Market Data, Price History, Volume, Metrics
- **Integration Models**: Integration Source, Connection State, Sync Status, Error States
- **Chain and Network Models**: Chain Identity, Network Configuration, Token Standards, Address Formats

These are **domain concepts** — they describe what the business cares about. A portfolio. An asset. A transaction. A chain. They form the ubiquitous language of CygnusWealth.

The specification also says data-models:
- **Does NOT own**: Business logic, data fetching, UI components, persistence
- Maintains **backward compatibility** as its primary stability guarantee
- Is depended upon by **every other bounded context** in the system

This is textbook DDD Shared Kernel: the minimal set of types that all bounded contexts agree on, kept maximally stable because changes propagate everywhere.

---

## 2. What the RPC Types Are

The en-25w5 types proposed for data-models:

| Type | What It Describes |
|------|-------------------|
| `RpcProviderRole` | Priority tier in a fallback chain (primary, secondary, tertiary, emergency) |
| `RpcProviderType` | Classification of an RPC relay service (managed, decentralized, public, community, user) |
| `RpcEndpointConfig` | A single RPC endpoint: URL, provider name, role, type, rate limit, timeout, weight |
| `ChainRpcConfig` | Per-chain RPC configuration: endpoint list, operation timeout, cache staleness thresholds |
| `RpcProviderConfig` | Top-level config: chain configs + circuit breaker + retry + health check settings |
| `CircuitBreakerConfig` | Failure threshold, window, recovery settings |
| `RetryConfig` | Max attempts, backoff delays, jitter |
| `HealthCheckConfig` | Ping interval, timeout, enabled flag |
| `UserRpcEndpoint` | User-provided RPC URL (en-tfkn) |
| `UserRpcConfig` | User override configuration with mode selection (en-tfkn) |
| `PrivacyConfig` | Provider rotation, privacy mode, query jitter (en-tfkn) |

These are **infrastructure configuration types**. They describe how the system communicates with blockchain nodes — not what a portfolio is, not what an asset is, not what a transaction looks like. No business stakeholder would recognize `CircuitBreakerConfig` as part of the domain language.

---

## 3. The DDD Case Against Placement in data-models

### 3.1 Shared Kernel Scope Violation

The DDD Shared Kernel pattern has a strict scope rule: include only what **all** bounded contexts need to agree on. The types that belong are identity types, domain events, and core value objects that appear in cross-context contracts.

Who actually consumes `RpcProviderConfig`?

| Bounded Context | Needs RPC Config Types? |
|----------------|------------------------|
| cygnus-wealth-app (Experience) | Yes — constructs the config as composition root |
| evm-integration (Integration) | Yes — receives config, creates fallback chains |
| sol-integration (Integration) | Yes — receives config, creates fallback chains |
| portfolio-aggregation (Portfolio) | **No** — calls integration interfaces, never sees RPC config |
| asset-valuator (Portfolio) | **No** — fetches prices from CoinGecko/CMC, different infrastructure |
| wallet-integration-system (Integration) | **No** — manages wallet connections, not RPC endpoints |
| coinbase-integration (Integration) | **No** — REST/OAuth API, not JSON-RPC |
| kraken-integration (Integration) | **No** — REST API with API key auth |
| robinhood-integration (Integration) | **No** — REST/OAuth API |

Three out of nine bounded contexts need these types. This is not "shared across all domains" — it's a contract between the app shell and two blockchain integration libraries. Putting it in the shared kernel forces the other six contexts to depend on (or at minimum, share a package that contains) types they will never use.

### 3.2 Stability Inversion

The shared kernel should be the most stable layer. Changes propagate to all consumers, so the change rate should be the lowest in the system.

RPC infrastructure configuration is inherently volatile:

- en-25w5 (2026-02-17) defined `RpcProviderType` as `managed | public | community`
- en-tfkn (2026-02-20, three days later) extended it to `managed | decentralized | public | community | user`
- en-tfkn also added `UserRpcConfig`, `PrivacyConfig`, and new fields to `RpcProviderConfig`

Three days between directives, and the types already changed. As CygnusWealth evolves — adding new provider tiers, new privacy features, method-level routing, advanced caching strategies — these types will keep changing. Each change to a type in data-models ripples to every consumer, requiring version bumps and compatibility checks across the entire system.

Compare this to the actual domain model types: `Portfolio`, `Asset`, `Balance`, `Transaction`. These change when the business model changes — which is rare. The domain model is stable. Infrastructure plumbing is not.

Placing volatile infrastructure types in the most stable layer inverts the stability gradient. This is a DDD anti-pattern.

### 3.3 Infrastructure Concern Contamination

DDD draws a hard line between domain model and infrastructure. The domain model expresses business concepts. Infrastructure is how those concepts are technically realized.

- "Ethereum is a chain with ID 1 and native token ETH" — **domain model** (belongs in data-models)
- "We reach Ethereum via an RPC endpoint at `eth-mainnet.g.alchemy.com` with a 5-second timeout and circuit breaker" — **infrastructure** (does not belong in data-models)

The first statement is true regardless of how we access Ethereum. The second is a deployment and runtime concern. If we switched from JSON-RPC to a GraphQL indexer tomorrow, the domain model wouldn't change — but every RPC infrastructure type would become obsolete.

By placing `CircuitBreakerConfig`, `RetryConfig`, and `HealthCheckConfig` in data-models, we embed infrastructure implementation details into the domain contract layer. The shared kernel should not know what a circuit breaker is.

### 3.4 The "Network Configuration" Trap

en-25w5 justified the placement by citing data-models.md:

> "The data-models.md already lists 'Network Configuration: RPC endpoints and chain parameters' as a planned model category"

This is true, but the listing was always ambiguous. "Network Configuration" in the domain model context means:

- **Domain-appropriate**: Chain ID, chain name, native token, block explorer URL, supported token standards, address format, network type (mainnet/testnet). These are identity and metadata about the chain itself.
- **Infrastructure-inappropriate**: RPC endpoint URLs, provider classifications, fallback priorities, circuit breaker thresholds, rate limits, retry policies. These are operational details about how we communicate with the chain.

The planned "Network Configuration" category in data-models should be fulfilled with chain identity and metadata types — not with RPC infrastructure configuration.

---

## 4. What Should Stay in data-models

These types legitimately belong in the shared kernel because they are domain identity concepts used across all contexts:

| Type | Why It Belongs |
|------|---------------|
| `ChainId` | Canonical chain identifier (e.g., `'ethereum'`, `'solana'`). Used everywhere — portfolio models reference chains, integrations identify which chain data comes from, the app displays chain info. This is ubiquitous language. |
| `ChainIdentity` | Chain metadata: name, native token symbol, native token decimals, block explorer URL, network type. Domain concept. |
| `SupportedChain` | Enumeration of chains CygnusWealth supports. Referenced by integration configs, portfolio models, and the app. |
| `AddressFormat` | Chain-specific address representation rules. Used by wallet integration and all blockchain integrations. |
| `TokenStandard` | ERC-20, ERC-721, ERC-1155, SPL. Cross-domain vocabulary. |

These are stable, truly shared, and part of the domain language.

---

## 5. Where RPC Types Should Live Instead

### Option A: Dedicated `rpc-infrastructure` Package (Recommended)

en-25w5 itself suggested this in Phase 3:

> "This could live in a shared `@cygnus-wealth/rpc-infrastructure` package"

The same package that houses the `CircuitBreaker` class, `RpcFallbackChain`, `TokenBucketRateLimiter`, and `HealthMonitor` implementations should also own the configuration types that parameterize them. The types and the code that consumes them belong together.

**Dependency graph with this approach:**

```
cygnus-wealth-app
    │
    ├──► rpc-infrastructure (types + runtime classes)
    │         │
    │         └──► data-models (ChainId, ChainIdentity — domain types only)
    │
    ├──► evm-integration
    │         │
    │         ├──► rpc-infrastructure
    │         └──► data-models
    │
    └──► sol-integration
              │
              ├──► rpc-infrastructure
              └──► data-models
```

**Why this works:**
- Types live next to the code that uses them
- Only the three contexts that need RPC config depend on this package
- data-models stays pure domain model
- rpc-infrastructure can evolve rapidly (new provider types, privacy features) without touching the shared kernel
- The package is still shared — no type duplication

### Option B: Integration Domain Published Language

Define the RPC configuration types as the integration domain's published language — the interface that external consumers (the app shell) program against when initializing integration libraries.

The types would live in a `@cygnus-wealth/integration-contracts` package or within each integration library's public API surface as a shared interface module.

**This is more DDD-orthodox** but creates a subtlety: the app shell (Experience domain) would depend on the Integration domain's published interface, which is already how the dependency graph works per contracts.md.

### Option C: Keep in data-models (Status Quo)

Accept the pragmatic argument: the types are pure, the system is small, and the cost of misplacement is a slight aesthetic violation. This is the least work but sets a precedent that data-models is a dumping ground for any shared type, which will compound as the system grows.

**Not recommended** for an architecture that explicitly aspires to DDD rigor.

---

## 6. Impact on en-25w5 and en-tfkn Implementation

### Phase 1 Redefinition

en-25w5 Phase 1 is currently: "Add the `RpcProviderConfig` type family to the Contract Domain."

Under the corrected placement, Phase 1 splits:

**Phase 1A — data-models: Chain Identity Types**
Add the domain-appropriate chain identity types that actually belong in the shared kernel: `ChainId`, `ChainIdentity`, `SupportedChain`, and related network metadata. These are referenced by portfolio models, integration configs, and the app. This is the "Network Configuration" fulfillment that data-models always intended.

**Phase 1B — rpc-infrastructure: RPC Configuration Types**
Create the `@cygnus-wealth/rpc-infrastructure` package with the `RpcProviderConfig` type family. This package will also receive the Phase 3 runtime classes (CircuitBreaker, FallbackChain, etc.), but the types can land first as a standalone foundation.

Phase 1A has zero dependencies. Phase 1B depends on Phase 1A (it references `ChainId` from data-models). Both can proceed quickly.

### en-tfkn Modifications Absorbed Cleanly

en-tfkn's additions (`RpcProviderType` extension, `UserRpcConfig`, `PrivacyConfig`) land in `rpc-infrastructure` instead of data-models. This is actually cleaner — en-tfkn extends infrastructure behavior, so the changes belong in the infrastructure package.

### No Structural Change to en-25w5 Phases 2-6

The factory signatures stay the same: `createEvmIntegration(config: RpcProviderConfig)`. The app bootstrap stays the same. The only difference is the import path: `from '@cygnus-wealth/rpc-infrastructure'` instead of `from '@cygnus-wealth/data-models'`.

---

## 7. The Counter-Argument (and Why It Fails)

**"But multiple bounded contexts need these types, and data-models is where shared types go."**

Not all shared types are shared kernel types. The shared kernel is for domain model types that represent the ubiquitous language. Types that are shared between a subset of contexts for infrastructure purposes are shared infrastructure — a different category with different stability requirements and a different place in the dependency graph.

DDD provides multiple sharing patterns:
- **Shared Kernel**: Core domain types all contexts agree on (Portfolio, Asset, ChainId)
- **Published Language**: Types a domain exports for consumers (integration factory interfaces)
- **Conformist / Anti-Corruption Layer**: One context adapts to another's types
- **Shared Library**: Technical utilities and infrastructure shared by multiple contexts

`RpcProviderConfig` fits the shared library / published language category, not the shared kernel.

**"The types are pure — no runtime behavior. What's the harm?"**

The harm is precedent and coupling direction. If `CircuitBreakerConfig` belongs in data-models because it's "just a type," then so does `OAuthConfig` for Coinbase, `ApiKeyConfig` for Kraken, `WebSocketConfig` for real-time subscriptions, and every other infrastructure configuration type. The shared kernel becomes an everything-bucket, and its stability guarantee — the reason it exists — erodes.

---

## 8. Recommendation

**Adopt Option A.** Create the `rpc-infrastructure` package to house both the configuration types (Phase 1B) and the runtime classes (Phase 3). The data-models shared kernel receives only domain-appropriate chain identity types (Phase 1A).

This corrects the placement before any code is written, at zero refactoring cost. The en-25w5 implementation plan requires only an import path change and a Phase 1 split. The en-tfkn modifications land more naturally in the infrastructure package where they belong.

---

## 9. Decision Summary

| Question | Answer |
|----------|--------|
| Was placing RPC types in data-models a mistake? | Yes, from strict DDD perspective. It conflates infrastructure config with domain model. |
| Are the types wrong? | No. The type definitions themselves are well-designed. Only their location is wrong. |
| Does this change the en-25w5 architecture? | No. Factory signatures, config flow, fallback chain behavior — all unchanged. Only the package that hosts the types changes. |
| Does this block en-tfkn implementation? | No. It simplifies it — en-tfkn's infrastructure extensions land in an infrastructure package instead of the domain contract layer. |
| What goes in data-models instead? | Chain identity types: ChainId, ChainIdentity, SupportedChain, AddressFormat, TokenStandard. |
| What's the implementation impact? | Phase 1 splits into 1A (data-models: chain identity) and 1B (rpc-infrastructure: RPC config types). Zero impact on Phases 2-6. |
| What if we ignore this and ship anyway? | The system works. The types are pure. But the shared kernel absorbs infrastructure volatility, and the precedent invites further contamination. The cost of fixing it now is near-zero; the cost of fixing it after 9 bounded contexts depend on a bloated data-models is significant. |
