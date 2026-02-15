# RPC Node Strategy

> **Navigation**: [Integration Domain](./README.md) > RPC Node Strategy

## Overview

This document defines the RPC provider strategy for CygnusWealth's Integration Domain, covering all supported EVM chains and Solana. The strategy prioritizes free/cheap tiers for development with clear scaling paths for production, and includes a multi-provider fallback architecture aligned with the domain's resilience principles.

**For context**: This strategy supports the [EVM Integration](./bounded-contexts/evm-integration.md) and [Solana Integration](./bounded-contexts/sol-integration.md) bounded contexts, and builds on the [Resilience & Performance Guide](./resilience-performance.md) fallback chain pattern.

## Provider Evaluation Summary

### Providers Evaluated

| Provider | Type | Free Tier | Chains | Strengths |
|----------|------|-----------|--------|-----------|
| **Alchemy** | Managed | 30M CU/mo | 100+ (EVM + Solana) | Best EVM tooling, reliable, Enhanced APIs |
| **Infura** | Managed | 3M credits/day | 20+ (EVM + Solana) | Mature, Ethereum-focused, MetaMask parent |
| **Ankr** | Decentralized | Public endpoints free, 200M credits on freemium | 40+ public, 80+ premium | Public RPCs, broad chain support |
| **dRPC** | Decentralized | 210M CU/mo | 180+ (EVM + Solana) | Most generous free tier, MEV protection (paid only) |
| **QuickNode** | Managed | 10M credits/mo | 80+ (EVM + Solana) | Enterprise-grade, SOC 2 / ISO 27001 certified |
| **Helius** | Managed (Solana) | 1M credits/mo | Solana only | Solana-native, best Solana tooling, Gatekeeper Edge |
| **Public RPCs** | Community | Free, no signup | Varies | No auth, instant access, rate-limited |

### Key Findings

1. **dRPC** offers the most generous free tier (210M CU/mo) with 180+ chains including Solana
2. **Alchemy** provides the best developer experience with 30M CU/mo free across 100+ chains; Growth/Scale tiers deprecated in favor of Pay-As-You-Go ($0.45/1M CU)
3. **Helius** is the clear winner for Solana with Gatekeeper Edge Gateway (Feb 2026) reducing latency 4-7x
4. **Infura** now supports 20+ chains including Base, Solana, and BNB (via opBNB L2); still no native Fantom support
5. **QuickNode** now offers a permanent free tier (10M credits/mo, 15 RPS) with SOC 2 Type II and ISO 27001 certifications
6. **Ankr** provides ungated public endpoints for 40+ chains at ~30 RPS; freemium tier adds 200M monthly credits

## Recommended Provider per Chain

### EVM Chains

#### Ethereum Mainnet

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Best Enhanced APIs, reliable infrastructure, archive data |
| **Secondary** | dRPC | Managed (API key) | Generous free tier, decentralized failover |
| **Tertiary** | Ankr | Public | No-auth fallback: `https://rpc.ankr.com/eth` |
| **Emergency** | Public | Community | `https://eth.llamarpc.com`, `https://1rpc.io/eth` |

**Dev/Testnet**: Alchemy free tier on Sepolia (primary), dRPC free tier (secondary)

#### Polygon (PoS)

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Strong Polygon support, Enhanced APIs |
| **Secondary** | dRPC | Managed (API key) | Broad coverage, low latency |
| **Tertiary** | Ankr | Public | `https://rpc.ankr.com/polygon` |
| **Emergency** | Public | Community | `https://polygon-rpc.com`, `https://1rpc.io/matic` |

**Dev/Testnet**: Alchemy free tier on Amoy testnet

#### Arbitrum One

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Excellent L2 support |
| **Secondary** | dRPC | Managed (API key) | Good Arbitrum performance |
| **Tertiary** | Infura | Managed (API key) | Arbitrum supported on free tier |
| **Emergency** | Public | Community | `https://arb1.arbitrum.io/rpc`, `https://1rpc.io/arb` |

**Dev/Testnet**: Alchemy free tier on Arbitrum Sepolia

#### Optimism

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Strong OP Stack support |
| **Secondary** | dRPC | Managed (API key) | Decentralized coverage |
| **Tertiary** | Infura | Managed (API key) | Optimism supported on free tier |
| **Emergency** | Public | Community | `https://mainnet.optimism.io`, `https://1rpc.io/op` |

**Dev/Testnet**: Alchemy free tier on OP Sepolia

#### BNB Smart Chain

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Recently added BNB support |
| **Secondary** | dRPC | Managed (API key) | Strong BSC coverage |
| **Tertiary** | Ankr | Public | `https://rpc.ankr.com/bsc` |
| **Emergency** | Public | Community | `https://bsc-dataseed.binance.org`, `https://1rpc.io/bnb` |

**Note**: Infura supports BNB ecosystem via opBNB (L2) only, not BSC mainnet directly. QuickNode supports BSC on all tiers including free.

**Dev/Testnet**: dRPC free tier on BSC Testnet, Ankr public endpoints

#### Avalanche C-Chain

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Good Avalanche support |
| **Secondary** | dRPC | Managed (API key) | Broad chain coverage |
| **Tertiary** | Infura | Managed (API key) | Avalanche C-Chain on free tier |
| **Emergency** | Public | Community | `https://api.avax.network/ext/bc/C/rpc`, `https://1rpc.io/avax/c` |

**Dev/Testnet**: Alchemy free tier on Fuji testnet

#### Base

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Alchemy | Managed (API key) | Strong OP Stack / Base support |
| **Secondary** | dRPC | Managed (API key) | Good Base performance |
| **Tertiary** | Infura | Managed (API key) | Base now supported on Infura |
| **Emergency** | Public | Community | `https://mainnet.base.org`, `https://1rpc.io/base` |

**Dev/Testnet**: Alchemy free tier on Base Sepolia

#### Fantom Opera

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | dRPC | Managed (API key) | Primary managed provider for Fantom |
| **Secondary** | Ankr | Public | `https://rpc.ankr.com/fantom` |
| **Tertiary** | Public | Community | `https://rpcapi.fantom.network`, `https://rpc.ftm.tools` |
| **Emergency** | Public | Community | `https://1rpc.io/ftm`, `https://fantom-mainnet.public.blastapi.io` |

**Note**: Alchemy has deprecated Fantom support. Infura does not support Fantom natively. QuickNode supports Fantom on all tiers including free. Consider monitoring Sonic (Fantom's successor chain) for future migration.

**Dev/Testnet**: dRPC free tier on Fantom Testnet, Ankr public endpoints

### Solana

| Role | Provider | Endpoint Type | Rationale |
|------|----------|--------------|-----------|
| **Primary** | Helius | Managed (API key) | Solana-native, best tooling, DAS API, staked connections by default on paid plans |
| **Secondary** | Alchemy | Managed (API key) | Multi-chain consistency, good Solana support |
| **Tertiary** | dRPC | Managed (API key) | Generous free tier, stake-weighted QoS on Solana, decentralized fallback |
| **Quaternary** | Infura | Managed (API key) | Solana now supported on Infura free tier |
| **Emergency** | Public | Community | `https://api.mainnet-beta.solana.com` (rate-limited) |

**Dev/Testnet**: Helius free tier on Devnet (primary), Alchemy free tier on Devnet (secondary)

**Solana-Specific Considerations**:
- Helius staked connections now included by default on all paid plans (1 credit/tx, down from 50)
- Helius Gatekeeper Edge Gateway (Feb 2026): 4.6x faster new connections, 7.8x faster reused connections
- Helius DAS API is essential for compressed NFTs and token metadata (2 RPS on free, up to 100 RPS on Professional)
- dRPC now supports stake-weighted QoS on Solana endpoints
- Public Solana RPC (`api.mainnet-beta.solana.com`) is heavily rate-limited; avoid relying on it
- Helius free tier (1M credits, 10 RPS) is sufficient for development
- For production, Helius Developer ($49/mo, 50 RPS) or Business ($499/mo, 200 RPS) tiers scale well

## Development Environment Strategy

### Free Tier Budget Allocation

For development and testing, the combined free tiers provide substantial capacity:

| Provider | Free Allocation | Primary Use |
|----------|----------------|-------------|
| dRPC | 210M CU/mo, 100 RPS | EVM chains primary dev provider |
| Alchemy | 30M CU/mo, ~25 RPS (500 CU/s) | Enhanced APIs, fallback for all chains |
| Helius | 1M credits/mo, 10 RPS | Solana development |
| QuickNode | 10M credits/mo, 15 RPS | Supplementary for all chains |
| Infura | 3M credits/day, 500 credits/s | Ethereum/L2/Solana supplementary |
| Ankr | 200M credits/mo (freemium), ~30 RPS | Emergency fallback, quick prototyping |

*Ankr public endpoints are rate-limited (~30 RPS) but freemium tier provides 200M monthly credits at no cost.

### Development Workflow

1. **Local Development**: Use dRPC free tier as primary, Alchemy as secondary
2. **CI/CD Testing**: Dedicated API keys on dRPC/Alchemy free tiers with test-specific rate limits
3. **Staging**: Same providers as development but with separate API keys for isolation
4. **Testnet Work**: All providers offer free testnet access; prefer Alchemy for consistent DX

### API Key Management

- Separate API keys per environment (dev, CI, staging, production)
- Store keys in environment variables, never in source code
- Rotate keys on a quarterly schedule
- Monitor usage per key to detect anomalies

## Production Scaling Strategy

### Tier Progression

```
Phase 1 (MVP / Low Traffic)
├── EVM: Alchemy free + dRPC free (combined ~240M CU/mo)
├── Solana: Helius free (1M credits/mo)
└── Cost: $0/mo

Phase 2 (Growing / Moderate Traffic)
├── EVM: Alchemy PAYG ($0.45/1M CU) + dRPC Growth ($0.30/1M CU)
├── Solana: Helius Developer ($49/mo, 10M credits, 50 RPS)
└── Estimated Cost: $50-150/mo

Phase 3 (Scale / High Traffic)
├── EVM: Alchemy PAYG (volume) + dRPC Growth (5K RPS)
├── Solana: Helius Business ($499/mo, 100M credits, 200 RPS)
├── Backup: QuickNode Scale ($499/mo, 950M credits, 250 RPS)
└── Estimated Cost: $500-1500/mo

Phase 4 (Enterprise / High Volume)
├── EVM: Alchemy Enterprise + dRPC Enterprise (custom SLA, unlimited RPS)
├── Solana: Helius Professional ($999/mo, 200M credits, 500 RPS) or Enterprise
├── Backup: QuickNode Business ($999/mo, 2B credits, 500 RPS)
└── Estimated Cost: $2000+/mo (negotiable)
```

### Production Rate Limits

Configure rate limiters per provider to stay within plan limits:

| Provider | Free Tier RPS | Paid RPS | Enterprise RPS |
|----------|--------------|----------|----------------|
| Alchemy | ~25 (500 CU/s) | ~300 (10K CU/s PAYG) | Custom |
| dRPC | 100 | 5,000 (Growth) | Unlimited |
| Helius | 10 | 50 (Dev) / 200 (Biz) | 500+ (Pro) |
| QuickNode | 15 | 50-250 (Build-Scale) | 500-100K+ |
| Infura | ~500 credits/s | 4K credits/s (Dev) | Elastic |
| Ankr (public) | ~30 | 1,500 (Premium PAYG) | Custom |

## Fallback Strategy

### Architecture

The fallback strategy follows the Integration Domain's [Fallback Chain Pattern](./resilience-performance.md#fallback-chain-pattern):

```
Request
  │
  ▼
┌─────────────────┐
│ Primary Provider │  ← Managed (Alchemy/Helius)
│ (Circuit Breaker)│
└────────┬────────┘
         │ failure
         ▼
┌─────────────────┐
│Secondary Provider│  ← Managed (dRPC/Alchemy)
│ (Circuit Breaker)│
└────────┬────────┘
         │ failure
         ▼
┌─────────────────┐
│Tertiary Provider │  ← Managed/Public (Infura/Ankr)
│ (Circuit Breaker)│
└────────┬────────┘
         │ failure
         ▼
┌─────────────────┐
│ Emergency Public │  ← Public RPCs (no auth)
│   (Best Effort) │
└────────┬────────┘
         │ failure
         ▼
┌─────────────────┐
│   Cached Data   │  ← Last known good (L2 cache)
│  (Stale OK)     │
└────────┬────────┘
         │ no cache
         ▼
┌─────────────────┐
│  Offline Mode   │  ← User notification
└─────────────────┘
```

### Failover Rules

1. **Circuit Breaker Thresholds**: 5 failures in 60 seconds triggers OPEN state (per provider, per chain)
2. **Timeout per attempt**: 5 seconds for managed providers, 10 seconds for public RPCs
3. **Total operation timeout**: 30 seconds across all fallback attempts
4. **Cache acceptance**: Serve stale cache up to 5 minutes for balances, 1 hour for metadata
5. **Health checks**: Ping all providers every 60 seconds; mark unhealthy providers as last resort

### Provider Health Monitoring

Each provider endpoint is monitored with:
- Latency tracking (P50, P95, P99)
- Error rate calculation (rolling 5-minute window)
- Availability scoring (weighted by recency)
- Automatic promotion/demotion based on performance

Providers that consistently outperform their assigned role can be dynamically promoted (e.g., if the secondary provider has lower latency than primary, swap them).

## Chain-Specific Considerations

### High-Value Chains (Ethereum, Polygon, Arbitrum)
- Highest traffic expected; allocate more of the rate limit budget
- Archive data access important for transaction history
- WebSocket connections preferred for real-time updates
- Use Alchemy Enhanced APIs for token metadata and transfers

### L2 Chains (Optimism, Base, Arbitrum)
- Lower RPC costs per request (simpler state)
- Finality timing differs from L1; adjust cache TTLs accordingly
- OP Stack chains (Optimism, Base) share similar RPC behavior

### Alternative L1s (BNB, Avalanche, Fantom)
- Fewer managed provider options (especially Fantom)
- Public RPCs are more reliable for these chains due to lower traffic
- Fantom's migration to Sonic may require provider reassessment

### Solana
- Fundamentally different RPC model from EVM (account-based queries, program accounts)
- Staked connections significantly improve transaction confirmation
- DAS (Digital Asset Standard) API needed for compressed NFTs
- `getProgramAccounts` calls are expensive; batch and cache aggressively
- Solana public RPC is unreliable for production; always use a managed provider

## Security Requirements

All RPC provider usage must adhere to the Integration Domain's security principles:

1. **Read-Only**: All RPC calls are read-only (eth_call, eth_getBalance, getAccountInfo, etc.)
2. **No Private Keys**: RPC endpoints never receive private key material
3. **API Key Protection**: Keys stored in environment variables, never committed to source
4. **Request Validation**: Validate all responses from RPC providers before passing to domain
5. **Privacy**: Minimize address exposure; batch queries where possible to reduce fingerprinting
6. **Rate Limit Compliance**: Respect provider rate limits to avoid service disruption

## Decision Log

| Decision | Rationale | Date |
|----------|-----------|------|
| Alchemy as primary EVM provider | Best DX, Enhanced APIs, 100+ chains, PAYG pricing ($0.45/1M CU) | 2026-02-14 |
| dRPC as secondary EVM provider | Most generous free tier (210M CU), 180+ chains, decentralized architecture | 2026-02-14 |
| Helius as primary Solana provider | Solana-native, DAS API, staked connections default on paid, Gatekeeper Edge (4-7x latency improvement) | 2026-02-14 |
| Infura as tertiary (expanded role) | Now supports 20+ chains including Base, Solana, and BNB (opBNB); still no native Fantom | 2026-02-14 |
| Fantom uses dRPC as primary | Alchemy deprecated Fantom; dRPC has active Fantom support across 180+ chains | 2026-02-14 |
| QuickNode as supplementary provider | Now has permanent free tier (10M credits/mo, 15 RPS); SOC 2 / ISO 27001 certified; good production backup | 2026-02-14 |
| Multi-provider fallback for all chains | Aligns with domain resilience principles; no single point of failure | 2026-02-14 |
| dRPC MEV protection paid-only | MEV protection on Ethereum, Base, BNB, Polygon, Solana requires paid tier | 2026-02-14 |

## Related Documentation

- **[<- Integration Domain README](./README.md)** - Domain overview and strategic guidance
- **[Resilience & Performance Guide](./resilience-performance.md)** - Fallback chain and circuit breaker patterns
- **[Integration Patterns Guide](./patterns.md)** - Connection pooling and rate limit management
- **[EVM Integration](./bounded-contexts/evm-integration.md)** - EVM chain-specific implementation
- **[Solana Integration](./bounded-contexts/sol-integration.md)** - Solana-specific implementation
