# en-tfkn: Decentralized RPC Provider Strategy

**Status**: ENTERPRISE-WIDE ARCHITECTURE DIRECTIVE
**Date**: 2026-02-20
**Bead**: en-tfkn
**Scope**: Enterprise-wide RPC provider selection, decentralized-first hierarchy, user-configurable endpoints, privacy-aligned provider strategy, cross-domain contract updates, phased implementation roadmap
**Builds On**: en-25w5 (RPC Provider Fallback Chain), rpc-strategy.md, patterns.md, resilience-performance.md

---

## Executive Summary

**Our integration rigs hardcode free public RPC endpoints that are unreliable and centralized. This directive replaces the centralized-first provider model with a decentralized-first strategy aligned with CygnusWealth's sovereignty principles.**

The immediate trigger: `rpc.ankr.com/solana` returning HTTP 403 in production, causing total Solana data loss. But the root problem is architectural — we default to centralized single-company endpoints (Alchemy, Infura) as primaries, with public RPCs as fallbacks. A single company's rate limit policy change or outage cascades through the entire system. This contradicts our core principle: no single entity should control the user's access to their own financial data.

This directive establishes a decentralized-first provider hierarchy where decentralized relay networks (POKT Network, dRPC, Lava Network) serve as primary and secondary providers. Centralized managed providers (Alchemy, Infura, Helius) become tertiary fallbacks. Users can override any of this with their own endpoints. No API keys are ever baked into the client bundle.

### Relationship to en-25w5

The en-25w5 directive defines **how** RPC configuration is structured, propagated, and consumed — the types (`RpcProviderConfig`, `ChainRpcConfig`, `RpcEndpointConfig`), the bootstrap flow from `cygnus-wealth-app` to integration libraries, the circuit breaker/fallback chain implementation, and the 6-phase implementation plan. All of that remains valid and unchanged.

This directive (en-tfkn) redefines **which** providers fill those configuration slots and **why**. It inverts the provider hierarchy from centralized-first to decentralized-first, adds user-configurable endpoint support, establishes privacy evaluation criteria, and extends the `RpcProviderType` taxonomy. The en-25w5 implementation plan absorbs these changes — no new phases are required, only updated provider defaults and a new `UserRpcConfig` type.

---

## 1. Problem Statement

### Current State

The rpc-strategy.md provider selection (and en-25w5's default chain configurations) places centralized managed providers as primary:

| Chain | Current Primary | Current Secondary |
|-------|----------------|-------------------|
| Ethereum | Alchemy (centralized) | dRPC (decentralized) |
| Polygon | Alchemy (centralized) | dRPC (decentralized) |
| Solana | Helius (centralized) | Alchemy (centralized) |
| All EVM | Alchemy (centralized) | dRPC (decentralized) |

This means:
- **Single-company dependency**: Alchemy serves as primary for 8 of 9 chains. An Alchemy-wide outage takes down primary access to all EVM data simultaneously.
- **API key exposure risk**: This is a client-side browser app. Any API key shipped in the bundle is public. Managed provider keys embedded client-side can be extracted, abused, and revoked — causing system-wide failure.
- **Privacy concentration**: Every balance query, every transaction lookup, every portfolio refresh flows through one company's infrastructure. Alchemy sees the complete financial picture of every user.
- **Centralization contradiction**: CygnusWealth's founding principle is client-side sovereignty. Routing all blockchain queries through a single company's servers contradicts this.

### The Incident

`rpc.ankr.com/solana` returned HTTP 403, blocking all Solana balance queries. With no fallback chain implemented yet (en-25w5 is unbuilt), this was a total outage for Solana. But Ankr is one of the more decentralized options — the problem would have been worse with a centralized provider whose terms of service changed.

### What Decentralized-First Means

A decentralized RPC relay network distributes queries across independent node operators. No single company, server, or jurisdiction controls the relay path. If one operator goes down, the network routes around it automatically at the protocol level — before our application-level circuit breakers even engage. This is defense in depth: protocol-level resilience underneath our application-level fallback chain.

---

## 2. Decentralized Provider Evaluation

### Evaluation Criteria

| Criterion | Weight | Why It Matters |
|-----------|--------|---------------|
| **Decentralization** | Critical | Core architectural principle — no single entity controls query routing |
| **Chain Coverage** | Critical | Must cover all 8 EVM chains + Solana |
| **Free Tier Capacity** | High | Client-side app cannot hide API keys; free/keyless access is the default mode |
| **Privacy** | High | Provider should not correlate all queries to build user financial profiles |
| **Reliability** | High | Low latency, high uptime, geographic distribution |
| **Keyless Access** | Medium | Ability to operate without any API key (reduces client-side exposure) |
| **Censorship Resistance** | Medium | No single operator can block queries |
| **Latency** | Medium | Sub-second response times for balance queries |

### Provider Deep Evaluation

#### POKT Network (Pocket Network)

**Architecture**: Fully decentralized relay protocol. Post-Shannon upgrade (June 2025), POKT is a permissionless coordination layer — any entity can run a gateway, any node operator can stake and serve relays. 25,000+ active nodes across 30+ countries.

**Decentralization Score**: Highest of all evaluated providers. No single company controls the gateway layer (post-Shannon). Multiple independent gateways compete for traffic. Node operators are economically incentivized via POKT token staking. Protocol-level fraud detection cross-references responses from multiple providers.

**Chain Coverage**: 60+ chains via the F-Chains program (free, no-key-required RPC access as a public good). Covers Ethereum, Polygon, Arbitrum, Optimism, BNB, Avalanche, Base, and Solana — all 9 of our target chains.

**Free Tier**: 1M relays/day through the Pocket Portal (no credit card required). F-Chains program provides keyless public RPC URLs for all supported chains. Service fees priced in compute units pegged to USD (1B CU = $1), making relay pricing 30-75% cheaper than centralized alternatives.

**Privacy**: Decentralized model means no single party sees all queries. Queries are distributed across thousands of independent operators. Users can run their own gateway for maximum privacy. However, the Pocket Portal (PNI-operated gateway) does log for operational purposes — the F-Chains public endpoints are the privacy-friendly option.

**Keyless Access**: Yes — F-Chains program provides keyless public RPC URLs. No API key needed for basic access.

**Limitations**: Latency can be higher than centralized providers due to relay routing. Some advanced RPC methods (debug/trace) may not be supported by all node operators. DAS API (Solana-specific) is not supported.

**Verdict**: **Primary provider for most chains.** Best decentralization, good chain coverage, keyless access, privacy-preserving architecture. Protocol-level resilience means failures are handled before our circuit breakers engage.

#### dRPC

**Architecture**: Hybrid decentralized. 50+ independent node operators within a unified routing fabric. 7 geo-distributed clusters with multi-region failover. Intelligent request routing based on latency, availability, and region. The routing layer itself is operated by dRPC (the company), but backend operators are independent.

**Decentralization Score**: Medium-high. The operator pool is genuinely independent (50+ operators), but the routing/gateway layer is company-controlled. This means dRPC-the-company could theoretically censor or throttle traffic, though their architecture distributes this risk across operators who independently decide their compliance stance.

**Chain Coverage**: 100+ chains, 190+ networks. The broadest coverage of any evaluated provider. All 9 target chains supported with excellent performance.

**Free Tier**: 210M compute units per 30-day period (most generous of any provider). Rate limit: 120,000 CU/minute per IP, ~2,100 CU/s dynamic limit. All free-tier traffic routes through public nodes only (less reliable than paid). Per-request cost after free tier: $0.30/1M CU (all methods flat 20 CU since June 2025).

**Privacy**: dRPC states they do not collect sensitive data beyond what's required for service delivery. Each operator independently decides their logging/compliance stance based on jurisdiction. The D-RPC Protocol's Phase 4 (Morpheus, 2027) plans Zero-Knowledge Proof integration for fully private, content-encrypted requests. Current state: better than centralized providers, but the routing layer sees traffic patterns.

**Keyless Access**: Free tier requires registration and API key. No keyless public endpoint option (unlike POKT F-Chains or Ankr public).

**Additional Features**: MEV protection (paid only, available on Ethereum, Base, BNB, Polygon, Solana). Debug and trace APIs. Team access controls. Enterprise SLAs.

**Limitations**: Free tier limited to public nodes (potentially slower/less reliable). Gateway is company-controlled (not fully permissionless). API key required even for free tier.

**Verdict**: **Secondary provider for all chains.** Most generous free tier, excellent chain coverage, good decentralization at the operator level. The 210M CU/month free tier provides substantial runway. API key requirement is manageable through the user-configurable endpoint system.

#### Lava Network

**Architecture**: Decentralized RPC routing protocol with on-chain verification. Providers stake LAVA tokens and are rewarded based on latency, freshness, and availability. "Specs" system allows permissionless addition of new chains. On-chain fraud detection cross-references responses to detect and penalize bad actors.

**Decentralization Score**: High. Protocol-level routing with on-chain enforcement. No single company controls the relay path. Provider competition is economically incentivized. Fraud detection is on-chain (not trust-based).

**Chain Coverage**: 30+ chains and rollups. Covers Ethereum, Polygon, Arbitrum, NEAR, Starknet, and others. Solana is supported. Coverage is growing via the permissionless "specs" system but is narrower than POKT or dRPC.

**Free Tier**: Freemium tier available with rate limits. Exact free tier limits are not publicly documented as generously as POKT or dRPC. $3.5M+ revenue generated since August 2024 suggests a more production/commercial focus.

**Privacy**: Strong architectural privacy — decentralized routing means no single provider sees all traffic. On-chain verification adds a transparency layer but also means some query metadata touches the chain. Provider diversity across jurisdictions.

**Keyless Access**: Freemium tier requires account creation. No keyless public endpoints documented.

**Additional Features**: SDK, Server Kit, and Gateway access options. Multi-access-point architecture. Performance-based provider selection (protocol rewards fast, fresh, available providers).

**Limitations**: Narrower chain coverage (30+ vs POKT's 60+ or dRPC's 100+). Fantom and some L2s may not be supported. Less documentation on free tier specifics. Smaller network effect than POKT.

**Verdict**: **Tertiary provider / fallback layer.** Good decentralization with on-chain enforcement, but narrower chain coverage limits its role as a primary. Best positioned as a fallback behind POKT and dRPC, especially for chains where Lava has strong operator presence.

### Evaluation Matrix

| Criterion | POKT Network | dRPC | Lava Network | Alchemy | Infura |
|-----------|-------------|------|-------------|---------|--------|
| **Decentralization** | Highest (permissionless) | Medium-High (50+ operators) | High (on-chain verified) | None (single company) | None (single company) |
| **Chain Coverage** | 60+ (all 9 targets) | 100+ (all 9 targets) | 30+ (most targets) | 100+ (all 9 targets) | 20+ (8 of 9 targets) |
| **Free Tier** | 1M relays/day + F-Chains keyless | 210M CU/month | Freemium (limited) | 30M CU/month | 3M credits/day |
| **Keyless Access** | Yes (F-Chains) | No (key required) | No (account required) | No (key required) | No (key required) |
| **Privacy** | Best (no single observer) | Good (operator diversity) | Good (on-chain verified) | Poor (single observer) | Poor (single observer) |
| **Censorship Resistance** | Best (permissionless gateways) | Good (operator autonomy) | Good (economic incentives) | None (corporate policy) | None (corporate policy) |
| **Latency** | Medium (relay routing) | Low (geo-distributed) | Medium (protocol routing) | Lowest (direct) | Low (direct) |
| **DAS API (Solana)** | No | No | No | No | No |
| **Advanced APIs** | Limited | Debug/Trace | Limited | Enhanced APIs | Limited |

---

## 3. Decentralized-First Provider Hierarchy

### The New Default Hierarchy

```
User-Provided Endpoint (if configured)
    │ not configured or failure
    ▼
Decentralized Primary (POKT Network)
    │ circuit breaker OPEN or failure
    ▼
Decentralized Secondary (dRPC)
    │ circuit breaker OPEN or failure
    ▼
Decentralized Tertiary (Lava Network)
    │ circuit breaker OPEN or failure
    ▼
Centralized Fallback (Alchemy / Helius / Infura)
    │ circuit breaker OPEN or failure
    ▼
Community Emergency (public RPCs)
    │ all fail
    ▼
Cached Data (stale-but-acceptable)
    │ no cache
    ▼
Offline Mode (user notification)
```

### Why This Order

1. **POKT as primary**: Best decentralization, keyless access via F-Chains, protocol-level resilience handles individual node failures before our circuit breakers engage. POKT's 25,000+ nodes across 30+ countries provide geographic redundancy that no single company can match.

2. **dRPC as secondary**: Most generous free tier (210M CU/month), 50+ independent operators, excellent performance with geo-distributed routing. If POKT has protocol-level issues, dRPC's independent operator network provides a different failure domain.

3. **Lava as tertiary**: On-chain verified routing with economic fraud prevention. Different protocol, different operator set, different failure domain. Adds a third decentralized layer before falling back to centralized providers.

4. **Centralized providers as fallback**: Alchemy, Helius, and Infura have the lowest latency and most advanced APIs (Enhanced APIs, DAS API). They serve as reliable fallbacks when decentralized networks have issues. Their centralized nature is acceptable in a fallback role — the user is already experiencing degradation.

5. **Community RPCs as emergency**: `llamarpc.com`, `1rpc.io`, chain-native endpoints. Last resort, heavily rate-limited, but available without any key or account.

### Solana Exception: Helius DAS API

Helius remains the primary provider for Solana DAS API methods (`getAssetsByOwner`, `getAssetsByGroup`, etc.) because no decentralized provider supports the DAS standard. For non-DAS Solana methods (balance queries, transaction history, account info), the decentralized-first hierarchy applies.

**Solana dual-path architecture**:

```
Solana RPC Call
    │
    ├─► DAS API method? (getAssetsByOwner, getAssetsByGroup, etc.)
    │       │
    │       ▼
    │   Helius (primary, only DAS provider)
    │       │ failure
    │       ▼
    │   Legacy method equivalent + cache
    │       │ failure
    │       ▼
    │   Cached DAS data (stale acceptable)
    │
    └─► Standard RPC method? (getBalance, getTransaction, etc.)
            │
            ▼
        Decentralized-first hierarchy (POKT → dRPC → Lava → Alchemy → public)
```

---

## 4. Updated Default Chain Configurations

### Extending RpcProviderType

The en-25w5 `RpcProviderType` taxonomy needs a new category:

**Current types**:
- `managed` — requires API key, has SLA/rate limits (Alchemy, Helius, QuickNode, Infura)
- `public` — authenticated but free-tier public endpoints (Ankr public RPCs)
- `community` — fully open, no auth, heavily rate-limited (llamarpc, 1rpc, chain-native)

**New type**:
- `decentralized` — distributed relay network, queries routed across independent operators, no single-company control (POKT F-Chains, dRPC, Lava Network)

This distinction matters because `decentralized` providers have protocol-level resilience — a single node operator failure doesn't require our circuit breaker to engage, the protocol handles it. Our circuit breakers only open for provider-wide issues (the entire POKT network degrading, not a single POKT node).

### Ethereum Mainnet

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | emergency | public (llamarpc) | community | 10 | 10000ms | No |
| 7 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### Polygon

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | emergency | public (polygon-rpc.com) | community | 10 | 10000ms | No |
| 7 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### Arbitrum One

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | fallback | Infura | managed | 15 | 5000ms | Yes |
| 7 | emergency | public (arb1.arbitrum.io) | community | 10 | 10000ms | No |
| 8 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### Optimism

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | fallback | Infura | managed | 15 | 5000ms | Yes |
| 7 | emergency | public (mainnet.optimism.io) | community | 10 | 10000ms | No |
| 8 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### BNB Smart Chain

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Alchemy | managed | 25 | 5000ms | Yes |
| 5 | fallback | Ankr | public | 30 | 5000ms | No |
| 6 | emergency | public (bsc-dataseed.binance.org) | community | 10 | 10000ms | No |
| 7 | emergency | public (1rpc) | community | 10 | 10000ms | No |

Note: Lava Network BNB support is unconfirmed. Ankr public serves as a decentralized-adjacent fallback.

### Avalanche C-Chain

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | fallback | Infura | managed | 15 | 5000ms | Yes |
| 7 | emergency | public (api.avax.network) | community | 10 | 10000ms | No |
| 8 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### Base

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) |
| 5 | fallback | Alchemy | managed | 25 | 5000ms | Yes |
| 6 | fallback | Infura | managed | 15 | 5000ms | Yes |
| 7 | emergency | public (mainnet.base.org) | community | 10 | 10000ms | No |
| 8 | emergency | public (1rpc) | community | 10 | 10000ms | No |

### Fantom Opera

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required |
|----------|------|----------|------|-----------|---------|-------------|
| 1 | user | User-provided | user | — | 5000ms | User's own |
| 2 | primary | POKT (F-Chains) | decentralized | 30 | 5000ms | No |
| 3 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) |
| 4 | tertiary | Ankr | public | 30 | 5000ms | No |
| 5 | emergency | public (rpcapi.fantom.network) | community | 10 | 10000ms | No |
| 6 | emergency | public (1rpc) | community | 10 | 10000ms | No |

Note: Neither Alchemy (deprecated Fantom) nor Lava (unconfirmed Fantom) serve as reliable fallbacks. dRPC + POKT provide the decentralized layer; Ankr public provides emergency access.

### Solana

| Priority | Role | Provider | Type | RPS Limit | Timeout | Key Required | DAS Support |
|----------|------|----------|------|-----------|---------|-------------|------------|
| 1 | user | User-provided | user | — | 5000ms | User's own | Unknown |
| 2 | primary (DAS) | Helius | managed | 10 | 5000ms | Yes | Yes |
| 3 | primary (standard) | POKT (F-Chains) | decentralized | 30 | 5000ms | No | No |
| 4 | secondary | dRPC | decentralized | 100 | 5000ms | Yes (free tier) | No |
| 5 | tertiary | Lava Network | decentralized | 20 | 5000ms | Yes (freemium) | No |
| 6 | fallback | Alchemy | managed | 25 | 5000ms | Yes | No |
| 7 | fallback | Infura | managed | 15 | 5000ms | Yes | No |
| 8 | emergency | public (api.mainnet-beta.solana.com) | community | 5 | 10000ms | No | No |

Note: Helius is the only DAS API provider and retains priority for DAS methods. For standard Solana RPC, the decentralized-first hierarchy applies. The public Solana RPC (`api.mainnet-beta.solana.com`) is the endpoint that triggered this directive — it remains as absolute last resort with explicit circuit breaker protection.

---

## 5. User-Configurable RPC Endpoints

### Design Principle

Users must be able to bring their own RPC endpoint for any chain. This is the sovereignty principle applied to infrastructure: the user controls where their queries go. A user running their own Solana validator, or paying for a dedicated Alchemy endpoint, or using a private POKT gateway should be able to override our defaults entirely.

### User Endpoint Configuration

#### New Type: `UserRpcConfig`

Add to `data-models` under Network Configuration, alongside the existing `RpcProviderConfig` types from en-25w5:

**`UserRpcEndpoint`** — a single user-provided endpoint:
- `chainId: string` — which chain this endpoint serves
- `url: string` — the HTTP(S) RPC endpoint URL
- `wsUrl?: string` — optional WebSocket endpoint URL
- `label?: string` — user-friendly name (e.g., "My Alchemy", "Home Node")

**`UserRpcConfig`** — the complete user override configuration:
- `endpoints: UserRpcEndpoint[]` — user-provided endpoints per chain
- `mode: 'override' | 'prepend'` — whether user endpoint replaces the entire fallback chain (`override`) or is inserted as highest priority with defaults as fallback (`prepend`)

#### Storage

User-provided RPC endpoints are stored in the encrypted local storage managed by `EncryptionService` (CygnusWealthApp), alongside other user configuration. They are NEVER sent to any server or stored unencrypted. If the user provides an endpoint URL containing an API key, that key is protected by the same AES-256 encryption as other sensitive data.

#### Validation

Before accepting a user-provided endpoint, the app performs:

1. **URL format validation**: Must be valid HTTPS URL (HTTP rejected with warning for non-localhost)
2. **Health check**: Send a lightweight RPC call (`eth_blockNumber` for EVM, `getHealth` for Solana) to verify the endpoint responds correctly
3. **Chain verification**: For EVM, call `eth_chainId` and verify it matches the expected chain ID. For Solana, call `getGenesisHash` to verify network
4. **Latency measurement**: Record initial latency as baseline for health monitoring

If validation fails, the endpoint is rejected with a descriptive error message. The user can retry or provide a different URL.

#### UI Integration

In the CygnusWealthApp settings panel:

- **Per-chain endpoint configuration**: Each supported chain shows current endpoint source (POKT, dRPC, etc.) and allows the user to provide a custom endpoint
- **Override mode selector**: "Replace all defaults" vs "Use as primary with defaults as fallback"
- **Health indicator**: Green/yellow/red showing endpoint health based on most recent health check
- **Test button**: Manually trigger a health check and latency measurement
- **Reset to defaults**: One-click removal of custom endpoint, returning to the decentralized-first default chain

#### How User Endpoints Integrate with the Fallback Chain

The `buildRpcProviderConfig(env, userConfig)` function (from en-25w5, extended to accept `UserRpcConfig`) handles merging:

**`prepend` mode** (default, recommended):
```
User Endpoint (highest priority)
    │ failure
    ▼
POKT (F-Chains) — decentralized primary
    │ failure
    ▼
dRPC — decentralized secondary
    │ failure
    ▼
[... rest of default chain ...]
```

**`override` mode** (for advanced users):
```
User Endpoint (only endpoint)
    │ failure
    ▼
Cached Data (stale-but-acceptable)
    │ no cache
    ▼
Offline Mode
```

In `override` mode, the user takes full responsibility. If their endpoint goes down, there is no fallback to default providers. This is appropriate for users running their own infrastructure who don't want any queries leaking to third-party providers.

---

## 6. No Baked-In API Keys

### The Client-Side Constraint

CygnusWealth is a client-side browser application. There is no backend server. This has a critical implication: **any API key included in the JavaScript bundle is public**. It can be extracted from browser DevTools, network inspection, or source map analysis in seconds.

### What This Means for Provider Strategy

1. **Keyless providers are prioritized**: POKT F-Chains and Ankr public endpoints require no API key. They can be hardcoded as defaults without security risk. This is why POKT F-Chains is the default primary — it provides authenticated, decentralized RPC access without exposing any key.

2. **Keyed providers are user-provided only**: dRPC, Lava, Alchemy, Helius, Infura — all require API keys. These keys are NEVER shipped in the bundle. They come from two sources:
   - **User-provided keys**: The user enters their own API key in settings, stored in encrypted local storage
   - **Environment variables for development**: `CYGNUS_RPC_*` env vars (per en-25w5) are for local development only, never bundled

3. **Default configuration works without any keys**: The system must be fully functional (if degraded) using only keyless providers. The default fallback chain for every chain must include at least one keyless endpoint before reaching the cache/offline fallback.

### Keyless Default Chain (Every Chain Must Have This)

```
POKT F-Chains (keyless, decentralized)
    │ failure
    ▼
Ankr Public (keyless, public)
    │ failure
    ▼
Community RPCs (keyless, heavily rate-limited)
    │ failure
    ▼
Cached Data
    │ no cache
    ▼
Offline Mode
```

This guarantees the app works on first launch without any configuration. Users who want better performance and reliability can add their own API keys for dRPC, Alchemy, Helius, etc. — but the app never requires them.

### Key Storage Architecture

When a user provides API keys for managed/keyed providers:

1. Keys are encrypted with the user's password-derived key (same as other sensitive data)
2. Stored in encrypted IndexedDB via `EncryptionService`
3. Decrypted at app initialization (after user authentication)
4. Injected into the `RpcProviderConfig` via the provider URL builder
5. Never written to `localStorage` (only IndexedDB with encryption)
6. Never included in IPFS backup payloads (keys are device-specific)
7. Cleared on logout if user selects "clear data on logout" option

---

## 7. Privacy Considerations

### Privacy Threat Model

A portfolio tracker queries blockchain nodes for balance and transaction data. Each query reveals:
- **Which addresses the user monitors**: The set of queries over a session reveals the user's complete portfolio composition
- **When the user checks their portfolio**: Query timing reveals activity patterns
- **Which chains the user cares about**: Query distribution reveals chain preferences
- **Asset changes over time**: Repeated queries show portfolio evolution

If all queries route through a single provider, that provider can build a complete financial profile of the user.

### Provider Privacy Assessment

| Provider | Query Logging | Single Observer | Jurisdiction | Privacy Rating |
|----------|--------------|-----------------|-------------|---------------|
| **POKT (F-Chains)** | Distributed across 25K+ nodes; no single operator sees all queries | No — queries distributed across multiple independent operators | Multi-jurisdictional (30+ countries) | Best |
| **dRPC** | Minimal collection per policy; operators independently decide logging | Routing layer sees traffic patterns; individual operators see their portion | Multi-jurisdictional (operator dependent) | Good |
| **Lava Network** | On-chain verification adds transparency; individual providers see their portion | No — protocol distributes across providers | Multi-jurisdictional | Good |
| **Ankr (public)** | Unknown logging on free public endpoints | Yes — Ankr operates all public nodes | Centralized | Medium |
| **Alchemy** | Full query logging for analytics, debugging, and billing | Yes — all queries visible to Alchemy | US-based (San Francisco) | Low |
| **Helius** | Full query logging for billing and analytics | Yes — all queries visible to Helius | US-based | Low |
| **Infura** | Full query logging; ConsenSys subsidiary | Yes — all queries visible to Infura/ConsenSys | US-based | Low |

### Privacy Mitigation Strategies

#### 1. Provider Rotation Within a Session

Instead of sending all queries for a chain to the same provider, rotate across available providers within the same priority tier. If POKT, dRPC, and Lava are all healthy for Ethereum, distribute queries across them:

- Balance query 1 → POKT
- Balance query 2 → dRPC
- Balance query 3 → Lava
- Balance query 4 → POKT (round-robin)

This prevents any single provider from seeing the complete query pattern. The rotation is within the same priority tier — it does not affect fallback ordering.

**Implementation**: Extend the `RpcFallbackChain` (from en-25w5) with a `rotateWithinTier` option. When enabled, endpoints within the same `RpcProviderRole` are selected via weighted round-robin instead of strict priority order. Only applicable when multiple endpoints in the same tier are healthy.

#### 2. Query Batching and Timing Obfuscation

- Batch multiple address queries into single multi-call requests where supported (already in en-25w5's batching strategy)
- Add random jitter (0-500ms) to non-user-initiated background refresh queries to prevent timing correlation
- Cache aggressively to reduce the total number of queries that leave the client

#### 3. User Privacy Controls

In CygnusWealthApp settings:

- **Provider rotation toggle**: Enable/disable intra-tier rotation (default: enabled)
- **Privacy mode**: When enabled, only uses decentralized providers (POKT, dRPC, Lava). Centralized providers (Alchemy, Helius, Infura) are removed from the fallback chain entirely. This reduces reliability but maximizes privacy
- **Self-hosted gateway option**: Advanced users can configure their own POKT gateway or dRPC node, ensuring queries never touch third-party infrastructure

---

## 8. Cross-Domain Contract Updates

### Changes to data-models (Contract Domain)

#### New Types

Add to the Network Configuration section (alongside en-25w5 types):

**`RpcProviderType` extension**: Add `'decentralized'` and `'user'` to the existing enum:
- `managed` — requires API key, centralized SLA (Alchemy, Helius, Infura, QuickNode)
- `decentralized` — distributed relay network, queries routed across independent operators (POKT, dRPC, Lava)
- `public` — authenticated but free-tier public endpoints (Ankr freemium)
- `community` — fully open, no auth, heavily rate-limited (llamarpc, 1rpc, chain-native)
- `user` — user-provided endpoint, unknown characteristics

**`UserRpcEndpoint`**: As defined in Section 5.

**`UserRpcConfig`**: As defined in Section 5.

**`PrivacyConfig`**: Privacy-specific RPC configuration:
- `rotateWithinTier: boolean` — distribute queries across providers in the same tier (default: true)
- `privacyMode: boolean` — restrict to decentralized providers only (default: false)
- `queryJitterMs: number` — random delay added to background queries (default: 250, range: 0-500)

#### Modified Types

**`RpcProviderConfig`** (from en-25w5): Add new fields:
- `privacy: PrivacyConfig` — privacy-related configuration
- `userOverrides?: UserRpcConfig` — user-provided endpoint overrides (injected at runtime after decryption)

### Changes to CygnusWealthApp (Experience Domain)

#### Configuration Bootstrap Extension

The `buildRpcProviderConfig(env, userConfig?)` function (from en-25w5 Phase 2) is extended:

1. Read `CYGNUS_RPC_*` env vars (development mode)
2. Read user-provided endpoints from encrypted storage (production mode)
3. For each chain, build the endpoint list:
   a. If user has configured an endpoint for this chain:
      - `prepend` mode: user endpoint first, then decentralized defaults, then managed defaults
      - `override` mode: user endpoint only
   b. If no user endpoint: POKT → dRPC → Lava → managed → community (per this directive)
4. Apply keyless-only filtering: if no API key is available for a keyed provider, skip it
5. Assemble the complete `RpcProviderConfig` with privacy defaults

#### New Settings UI Components

- RPC endpoint configuration per chain (in Settings → Advanced → RPC Providers)
- Privacy mode toggle (in Settings → Privacy)
- Provider rotation toggle (in Settings → Privacy)
- Provider health dashboard (in Settings → Advanced → RPC Providers)

### Changes to Integration Libraries

#### evm-integration

- `createEvmIntegration(config: RpcProviderConfig)` — no interface change, but the config now contains decentralized providers as primaries
- `RpcFallbackChain` gains `rotateWithinTier` behavior when `privacy.rotateWithinTier` is true
- Provider URL builder gains POKT F-Chains and Lava Network URL templates

#### sol-integration

- `createSolIntegration(config: RpcProviderConfig)` — no interface change
- DAS API path unchanged (Helius remains primary for DAS)
- Standard Solana RPC path now routes through decentralized providers first
- POKT F-Chains Solana URL template added

### Changes to Portfolio Aggregation

No changes. Portfolio aggregation consumes integration library interfaces, not RPC configuration. The provider hierarchy is transparent to the portfolio layer.

---

## 9. Performance and Reliability Targets

### Latency Targets

| Operation | Target (P50) | Target (P95) | Max Acceptable |
|-----------|-------------|-------------|----------------|
| Balance query (single chain) | 300ms | 800ms | 2000ms |
| Balance query (all 9 chains, parallel) | 500ms | 1200ms | 3000ms |
| Transaction history (single chain) | 400ms | 1000ms | 2500ms |
| Health check (single endpoint) | 200ms | 500ms | 5000ms |
| Endpoint validation (user-provided) | 500ms | 1500ms | 5000ms |

### Latency Implications of Decentralized-First

Decentralized relay networks add 50-200ms latency compared to direct centralized provider calls due to relay routing overhead. This is acceptable because:

1. **Cache first**: Most queries hit L1/L2 cache (target 70%+ hit ratio). Relay latency only applies to cache misses.
2. **Background refresh**: Non-user-initiated queries (background balance refresh, health checks) are not latency-sensitive.
3. **User-initiated queries**: For explicit "refresh" actions, 500ms P50 is well within acceptable UX thresholds.
4. **Fallback speed**: If the decentralized primary is slow, circuit breakers detect it within seconds and route to faster fallbacks.
5. **Privacy tradeoff**: Users who prioritize privacy accept slightly higher latency. Users who prioritize speed can provide their own Alchemy/Helius key.

### Reliability Targets

| Metric | Target |
|--------|--------|
| Data availability (at least one provider responds per chain) | 99.9% |
| Data freshness (balance data < 5 minutes old) | 99.5% |
| Fallback switch time (primary → secondary) | < 100ms (skip, not wait) |
| Circuit breaker detection time (failing endpoint) | < 60 seconds (5 failures within window) |
| Recovery detection time (endpoint restored) | < 60 seconds (next health check cycle) |
| Full chain failure → cache fallback time | < 30 seconds (totalOperationTimeoutMs) |

### Fallback Chain Performance Budget

Total operation timeout: 30 seconds across all providers. Budget allocation:

```
User endpoint (if configured):    5s timeout, 1 retry  = ~6s max
POKT (decentralized primary):     5s timeout, 1 retry  = ~6s max
dRPC (decentralized secondary):   5s timeout, 1 retry  = ~6s max
Lava (decentralized tertiary):    5s timeout, 0 retry  = ~5s max
Alchemy (centralized fallback):   5s timeout, 0 retry  = ~5s max
Community (emergency):           10s timeout, 0 retry  = ~10s max
```

In practice, circuit breakers cause immediate skipping (0ms, not timeout) for known-failing endpoints, so the real-world fallback sequence is much faster than the worst-case budget.

### Health Check Budget

Per en-25w5, health checks run every 60 seconds. With the expanded provider list:

- Up to 4 decentralized + 2 managed + 2 community per EVM chain = ~8 endpoints/chain
- 8 EVM chains × 8 endpoints = 64 health checks
- Solana: ~8 endpoints = 8 health checks
- Total: ~72 health checks per 60-second cycle = ~1.2 checks/second

This is negligible on all free tiers. POKT F-Chains health checks don't consume rate limits (keyless). dRPC's 210M CU/month can absorb 72 × 20 CU × 1440 min/day = ~2M CU/day for health checks alone — less than 1% of the monthly budget.

---

## 10. Phased Implementation Roadmap

This directive does NOT create new implementation phases — it modifies the inputs to the existing en-25w5 phases. The 6-phase plan from en-25w5 remains the implementation vehicle.

### Phase 1 Modifications (data-models types)

**Add**:
- `'decentralized'` and `'user'` to `RpcProviderType`
- `UserRpcEndpoint` type
- `UserRpcConfig` type
- `PrivacyConfig` type
- `privacy` and `userOverrides` fields to `RpcProviderConfig`

### Phase 2 Modifications (app config bootstrap)

**Add**:
- POKT F-Chains URL templates per chain (keyless, no env var needed)
- Lava Network URL templates per chain
- Updated provider hierarchy: decentralized first in default chain configs
- `buildRpcProviderConfig(env, userConfig?)` extended to accept user overrides
- User endpoint storage/retrieval from encrypted IndexedDB
- Endpoint validation logic (chain ID verification, health check)
- Keyless-only default mode (works without any env vars)

**Modify**:
- Default chain configurations use POKT as primary, dRPC as secondary (per Section 4)
- Provider URL builder gains POKT and Lava URL templates

### Phase 3 Modifications (RPC infrastructure)

**Add**:
- `rotateWithinTier` option in `RpcFallbackChain`
- Query jitter for background requests
- `'decentralized'` provider type handling in circuit breaker (higher failure threshold since protocol handles individual node failures)

**Modify**:
- Circuit breaker thresholds for decentralized providers: raise `failureThreshold` to 8 (from 5) and `failureWindowMs` to 120000 (from 60000). Rationale: decentralized providers handle individual node failures at the protocol level — our circuit breaker should only open for systemic provider-wide issues, not transient relay routing fluctuations.

### Phase 4 & 5 Modifications (integration lib wiring)

**Add**:
- POKT F-Chains as first default endpoint in EVM chain configs
- Lava Network endpoint support
- DAS-aware dual-path for Solana (Helius for DAS, decentralized for standard)

**Remove**:
- Any hardcoded Alchemy/Infura endpoints as primary defaults
- Any bundled API keys (assert at build time that no `CYGNUS_RPC_*` values are in the production bundle)

### Phase 6 Modifications (app wiring)

**Add**:
- Settings UI for per-chain endpoint configuration
- Privacy settings (rotation toggle, privacy mode)
- Provider health dashboard showing which providers are active/healthy per chain
- First-launch experience: app works immediately with keyless POKT defaults, prompts user to add keys for enhanced performance (optional)

### New: Build-Time Assertion

Add a build-time check (CI/CD step) that scans the production bundle for any `CYGNUS_RPC_*` environment variable values or known API key patterns. If found, the build fails. This prevents accidental key leakage into the client bundle.

---

## 11. Risk Assessment

| Risk | Severity | Mitigation |
|------|----------|-----------|
| POKT F-Chains latency higher than Alchemy | Medium | Cache-first architecture absorbs most queries. Users who need low latency can add their own Alchemy key. Health monitoring detects persistent latency issues and can demote. |
| POKT F-Chains rate limits insufficient for heavy users | Medium | dRPC's 210M CU/month free tier provides massive headroom as secondary. Per-chain fallback ensures no single chain is starved. |
| Lava Network chain coverage gaps (some chains unsupported) | Low | Lava is tertiary — if unsupported for a chain, skip to managed fallback. Chain coverage verified at build time against known support list. |
| dRPC free tier changes (e.g., June 2025 free plan update) | Medium | Multiple decentralized providers means no single provider's pricing change is catastrophic. User-configurable endpoints provide escape hatch. |
| All decentralized providers degraded simultaneously | Low | Centralized providers (Alchemy, Helius, Infura) remain in the fallback chain. Cache provides offline resilience. This scenario requires multiple independent protocol failures simultaneously. |
| User-provided endpoints serving malicious data | Medium | All RPC responses are validated through the anti-corruption layer (per patterns.md). Chain ID verification prevents wrong-chain attacks. Response schema validation catches malformed data. |
| Privacy mode reduces reliability (fewer providers) | Medium | Privacy mode is opt-in with clear UX warning about reduced fallback options. Default mode balances privacy and reliability. |
| POKT Shannon upgrade introduces breaking changes | Low | F-Chains public URLs are stable; protocol upgrades are transparent to consumers. Monitor POKT changelog as part of dependency management. |
| Decentralized providers block specific RPC methods | Medium | Circuit breaker's method-level tracking (if a provider consistently fails `debug_traceTransaction`, mark it as DAS-like restricted for that method). Fallback to managed providers for advanced methods. |

---

## 12. Decision Log

| Decision | Rationale | Date |
|----------|-----------|------|
| POKT Network as default primary for all EVM + standard Solana | Highest decentralization, keyless F-Chains access, 25K+ nodes, multi-jurisdiction, protocol-level resilience | 2026-02-20 |
| dRPC as default secondary | Most generous free tier (210M CU), 50+ independent operators, 100+ chains, best balance of decentralization and performance | 2026-02-20 |
| Lava Network as default tertiary | On-chain verified routing, different failure domain from POKT/dRPC, economic fraud prevention, growing chain support | 2026-02-20 |
| Alchemy/Helius/Infura demoted to fallback | Centralized single-company control contradicts sovereignty principle; acceptable as fallback, not primary | 2026-02-20 |
| Helius retains primary for Solana DAS API | No decentralized provider supports DAS; Helius is the only option for compressed NFTs and advanced token metadata | 2026-02-20 |
| User endpoints always highest priority | Sovereignty principle: user controls their infrastructure. Override mode for maximum sovereignty, prepend mode for convenience | 2026-02-20 |
| No API keys in production bundle | Client-side app means any key is public. Build-time assertion prevents accidental leakage | 2026-02-20 |
| Keyless defaults must be functional | First-launch experience must work without any configuration. POKT F-Chains + Ankr public + community RPCs provide this | 2026-02-20 |
| Provider rotation within tier for privacy | Prevents any single provider from seeing complete query pattern. Default enabled, user-configurable | 2026-02-20 |
| Higher circuit breaker thresholds for decentralized providers | Protocol-level resilience handles individual node failures; our circuit breakers should only detect systemic issues | 2026-02-20 |
| No new implementation phases | This directive modifies inputs to existing en-25w5 phases, not the phases themselves. Minimizes implementation disruption | 2026-02-20 |

---

## 13. Relationship to Existing Architecture Documents

| Document | Current Role | What This Directive Changes |
|----------|-------------|---------------------------|
| **rpc-strategy.md** | Provider evaluation, per-chain selection, fallback tiers | Provider hierarchy inverted (decentralized-first). POKT, Lava added as evaluated providers. Alchemy/Infura demoted from primary to fallback |
| **en-25w5** | Implementation specification: types, config flow, fallback chain, circuit breakers | Types extended (new RpcProviderType values, UserRpcConfig, PrivacyConfig). Default configs updated. Circuit breaker thresholds adjusted for decentralized providers. No structural changes |
| **patterns.md** | Circuit breaker, retry, token bucket, connection pool patterns | `rotateWithinTier` behavior added to fallback chain pattern. No pattern changes |
| **resilience-performance.md** | FallbackChain, HealthMonitor, BulkheadManager patterns | HealthMonitor extended for POKT-specific health checks. No structural changes |
| **evm-integration.md** | EVM chain RPC management | Default endpoints change from Alchemy-first to POKT-first. Interface unchanged |
| **sol-integration.md** | Solana RPC management, DAS API | Dual-path architecture: DAS stays Helius, standard goes decentralized-first. DAS exception formally documented |
| **data-models.md** | Network Configuration model category | New types added. Existing types extended |
| **cygnus-wealth-app.md** | App shell, configuration, settings | New settings panels for RPC endpoints and privacy. Config bootstrap extended |
| **contracts.md** | Inter-domain contracts | No contract changes. Provider hierarchy is internal to integration libs |

No existing architectural decision is reversed. Alchemy/Helius/Infura remain in the system — they move from primary to fallback roles. The implementation specification from en-25w5 is extended, not replaced. The sovereignty principle that motivated the original architecture is now consistently applied to infrastructure provider selection.
