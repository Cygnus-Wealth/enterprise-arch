# en-9qon: Cross-Chain DeFi Architecture Recommendation

**Status**: RECOMMENDATION
**Date**: 2026-02-17
**Bead**: en-9qon
**Scope**: DeFi integration structure, multi-chain DeFi scope, po-e31 guidance

---

## Executive Summary

**Recommendation: Option A — DeFi stays in chain-specific integration libraries.**

DeFi protocol integration is fundamentally chain-specific data acquisition. It uses the same RPC connections, circuit breakers, caching, and anti-corruption layers as token balance fetching. The cross-chain DeFi abstraction belongs in `data-models` (type definitions) and `portfolio-aggregation` (orchestration), not in a separate integration library.

po-e31 (DeFi aggregation in portfolio-aggregation) should be designed multi-chain from the start by consuming a uniform DeFi position interface that each chain-specific integration implements.

No new repositories are needed. No dependency direction changes required.

---

## Analysis

### Question 1: DeFi Scope Beyond EVM

#### Current State
- **evm-integration**: ev-82k added Beefy and Aave adapters. The Integration Domain README already lists "DeFi position discovery" as a key evm-integration capability. The spec mentions "smart contract event monitoring" and "contract interaction logs."
- **sol-integration**: Already handles Serum DEX market data, Raydium liquidity pools, and staking program balances under "Program Integration." Lists "Advanced DeFi position tracking" as a future enhancement.
- **SUI**: No integration rig exists. SUI DeFi (Cetus, Turbos, Scallop) cannot be scoped without a `sui-integration` bounded context.

#### Recommendation on Scope

**Wave 4 (po-e31) should target EVM DeFi and Solana DeFi.** SUI should be deferred to Wave 5 or later.

Rationale:
- EVM DeFi is already partially implemented (Beefy, Aave in evm-integration)
- Solana DeFi infrastructure partially exists (Serum, Raydium program parsing in sol-integration)
- SUI requires a net-new `sui-integration` bounded context — a prerequisite that doesn't yet exist. Designing po-e31 for SUI without `sui-integration` means speculating about interfaces
- po-e31 should be designed with a chain-agnostic DeFi position interface so SUI slots in when `sui-integration` is created, but it should not block on SUI

**Solana DeFi protocols to add to sol-integration scope (Wave 4)**:
- Marinade Finance (liquid staking — mSOL)
- Raydium (already partially covered — needs LP position tracking, not just market data)
- Jupiter (aggregator — position tracking for limit orders, DCA, perps)
- Orca (concentrated liquidity positions — Whirlpools)

**SUI DeFi protocols for future sui-integration (Wave 5+)**:
- Cetus (concentrated liquidity)
- Turbos Finance (perpetuals, LP)
- Scallop (lending/borrowing)

---

### Question 2: Structure — Option A vs Option B

#### Option A: DeFi in Chain-Specific Integration Libs

Each chain-specific integration library (evm-integration, sol-integration, future sui-integration) owns its chain's DeFi protocol adapters alongside token/balance fetching.

#### Option B: Separate defi-integration Bounded Context

A standalone `defi-integration` library that is chain-aware but separate from chain-specific packages.

#### Decision: Option A

**Six decisive reasons:**

**1. DeFi position reading IS blockchain data acquisition.**

Fetching a Beefy vault position reads EVM smart contract state via the same RPC connection used for token balances. Fetching a Marinade mSOL position reads Solana program accounts via the same RPC connection used for SPL balances. The data acquisition mechanism is identical — only the contract/program being read differs. There is no technical justification for a separate data acquisition boundary when the transport, connection management, error handling, and caching infrastructure is the same.

**2. Protocol adapters are irreducibly chain-specific.**

A Beefy vault adapter must decode EVM ABI responses. A Marinade adapter must deserialize Solana program account data using Borsh. An Orca adapter must parse Whirlpool tick arrays from Solana accounts. These share zero implementation code. The adapter layer — which is where all the complexity lives — cannot be made cross-chain. The only thing that is cross-chain is the *output type* (a DeFi position), and that abstraction already lives in `data-models`.

**3. Option B creates a dependency direction violation or duplication.**

If `defi-integration` is a peer of `evm-integration` in the Integration layer:
- It needs EVM RPC connections to call Beefy/Aave contracts → must depend on `evm-integration` → **violates "no cross-integration dependencies"**
- OR it duplicates RPC connection management, circuit breakers, retry logic, rate limiting → **violates DRY and creates configuration drift**
- OR it wraps `evm-integration` → becomes an orchestration layer, which is `portfolio-aggregation`'s job

There is no clean dependency path for Option B that doesn't violate an existing architectural constraint.

**4. Existing precedent is already Option A and working.**

ev-82k placed Beefy and Aave adapters in `evm-integration`. sol-integration already has Serum and Raydium under "Program Integration." This is the natural home — the pattern is established and the work has begun.

**5. The cross-chain abstraction already has a home.**

The thing that IS cross-chain — the concept of "a DeFi position" (vault balance, lending position, LP share, staking position) — is a data model concern. `data-models` already lists "DeFi positions" as portfolio content. The unified types for DeFi positions (`DeFiPosition`, `VaultPosition`, `LendingPosition`, `LiquidityPosition`, `StakingPosition`) belong in `data-models`, not in a separate integration library. This is where the cross-chain semantics are standardized.

**6. Orchestration of multi-chain DeFi positions already has a home.**

`portfolio-aggregation` already orchestrates parallel fetching from evm-integration, sol-integration, and robinhood-integration. Adding DeFi position aggregation (po-e31) is a natural extension of its existing responsibility — "Coordinates parallel data fetching from multiple integration domains" and its portfolio view already includes "DeFi positions." This is the correct place for the cross-chain DeFi *aggregation* logic.

---

## Architectural Recommendation

### Layer Responsibilities for DeFi

```
┌──────────────────────────────────────────────────────────────┐
│                    data-models (Contract)                      │
│  DeFiPosition, VaultPosition, LendingPosition,               │
│  LiquidityPosition, StakingPosition                          │
│  DeFiProtocol enum, DeFiPositionType enum                    │
│  Chain-agnostic type definitions only                         │
└──────────────────────────┬───────────────────────────────────┘
                           │ depends on
         ┌─────────────────┼─────────────────┐
         │                 │                 │
         ▼                 ▼                 ▼
┌────────────────┐ ┌────────────────┐ ┌────────────────┐
│ evm-integration│ │ sol-integration│ │ sui-integration│
│                │ │                │ │   (future)     │
│ Protocol       │ │ Protocol       │ │ Protocol       │
│ Adapters:      │ │ Adapters:      │ │ Adapters:      │
│  - Beefy       │ │  - Marinade    │ │  - Cetus       │
│  - Aave        │ │  - Raydium     │ │  - Turbos      │
│  - Uniswap     │ │  - Jupiter     │ │  - Scallop     │
│  - Compound    │ │  - Orca        │ │                │
│  - Lido        │ │                │ │                │
│                │ │                │ │                │
│ Uses existing  │ │ Uses existing  │ │ Own RPC        │
│ RPC, cache,    │ │ RPC, cache,    │ │ connections    │
│ circuit breaker│ │ circuit breaker│ │                │
└────────┬───────┘ └────────┬───────┘ └────────┬───────┘
         │                  │                   │
         │   All return DeFiPosition[]          │
         └──────────┬───────┘───────────────────┘
                    ▼
┌──────────────────────────────────────────────────────────────┐
│              portfolio-aggregation (Portfolio)                 │
│  po-e31: DeFi Aggregation                                    │
│    - Collects DeFiPosition[] from all chain integrations     │
│    - Deduplicates (e.g., bridged positions)                  │
│    - Reconciles cross-chain DeFi (same protocol, multi-chain)│
│    - Enriches with pricing via asset-valuator                │
│    - Exposes unified DeFi portfolio view                     │
└──────────────────────────────────────────────────────────────┘
```

### Internal Structure Within Each Integration Lib

Each chain-specific integration library should organize DeFi protocol adapters as a submodule:

```
evm-integration/
├── src/
│   ├── services/
│   │   ├── BalanceService.ts          # existing
│   │   ├── TransactionService.ts      # existing
│   │   ├── TrackingService.ts         # existing
│   │   └── DeFiService.ts            # NEW - facade for DeFi positions
│   ├── defi/
│   │   ├── index.ts                   # DeFi adapter registry
│   │   ├── types.ts                   # Internal adapter interface
│   │   ├── adapters/
│   │   │   ├── beefy-adapter.ts       # Beefy vault positions
│   │   │   ├── aave-adapter.ts        # Aave lending positions
│   │   │   ├── uniswap-adapter.ts     # Uniswap LP positions
│   │   │   └── lido-adapter.ts        # Lido staking positions
│   │   └── utils/
│   │       ├── abi-decoder.ts         # Shared ABI decoding
│   │       └── multicall.ts           # Batched contract reads
│   └── ...
```

```
sol-integration/
├── src/
│   ├── services/
│   │   ├── BalanceService.ts
│   │   ├── NFTService.ts
│   │   ├── TransactionService.ts
│   │   └── DeFiService.ts            # NEW
│   ├── defi/
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── adapters/
│   │   │   ├── marinade-adapter.ts
│   │   │   ├── raydium-adapter.ts
│   │   │   ├── jupiter-adapter.ts
│   │   │   └── orca-adapter.ts
│   │   └── utils/
│   │       └── account-parser.ts      # Program account deserialization
│   └── ...
```

### Contract Changes Required in data-models

New types needed (Extended tier — expedited RFC):

- `DeFiPositionType` enum: `vault`, `lending_supply`, `lending_borrow`, `liquidity_pool`, `staking`, `farming`, `perp_position`
- `DeFiProtocol` enum: `beefy`, `aave`, `uniswap`, `compound`, `lido`, `marinade`, `raydium`, `jupiter`, `orca`, `cetus`, `turbos`, `scallop`
- `DeFiPosition` interface: core position with `type`, `protocol`, `chain`, `underlyingAssets[]`, `value`, `apy`, `rewards[]`, `metadata`
- `VaultPosition` extends `DeFiPosition`: auto-compound vault specifics
- `LendingPosition` extends `DeFiPosition`: supply/borrow rates, collateral factor, health factor
- `LiquidityPosition` extends `DeFiPosition`: token pair, price range (for concentrated liquidity), fee tier, impermanent loss
- `StakingPosition` extends `DeFiPosition`: lock period, validator/operator, rewards accrued

These types define the cross-chain abstraction. Every chain-specific adapter transforms protocol-specific data into these types.

### Contract Change: BlockchainIntegrationContract Extension

The existing `BlockchainIntegrationContract` in contracts.md gains one method:

```
BlockchainIntegrationContract {
  // Existing
  getBalances(addresses): BalanceList
  getTransactions(addresses, options): TransactionList
  getTokens(addresses): TokenList
  getNFTs(addresses): NFTList
  subscribeToUpdates(addresses): Subscription

  // NEW
  getDeFiPositions(addresses): DeFiPositionList
}
```

Each chain integration implements `getDeFiPositions()` by invoking its chain-specific protocol adapters and returning unified `DeFiPosition[]`.

### po-e31 Design Guidance (Multi-Chain from Start)

po-e31 in `portfolio-aggregation` should:

1. **Call `getDeFiPositions()` on all chain integrations in parallel** — same scatter-gather pattern already used for `getBalances()`
2. **Deduplicate cross-chain positions** — same protocol on multiple chains (e.g., Aave on Ethereum + Aave on Arbitrum) should be grouped but shown per-chain
3. **Aggregate DeFi value into portfolio totals** — sum DeFi position values into overall portfolio
4. **Request pricing for underlying assets** via `asset-valuator` — LP tokens, vault shares need to be priced by their underlying composition
5. **Apply partial failure strategy** — if sol-integration DeFi fails, EVM DeFi positions are still returned with failure metadata
6. **Expose DeFi positions as first-class portfolio content** alongside crypto holdings, TradFi positions, and NFTs

po-e31 does NOT need to know about Beefy, Aave, or Marinade specifically. It only sees `DeFiPosition[]` from each integration. The protocol-specific knowledge is fully encapsulated in the chain integrations.

This makes po-e31 automatically multi-chain from the start — and automatically supports SUI DeFi when `sui-integration` implements `getDeFiPositions()`.

---

## Dependency Direction (Unchanged)

```
cygnus-wealth-app
    │
    ▼
portfolio-aggregation ──────► wallet-integration-system
    │
    ├──► evm-integration ──► data-models
    ├──► sol-integration ──► data-models
    ├──► (sui-integration) ► data-models   [future]
    ├──► robinhood-integration ──► data-models
    └──► asset-valuator ──► data-models
```

No new dependency arrows. No cross-integration dependencies. The DeFi position types in `data-models` are consumed by integration libs (for output) and portfolio-aggregation (for input). This is the existing pattern.

---

## What Needs to Happen (Wave 4-5 Sequencing)

### Wave 4: EVM + Solana DeFi

1. **data-models**: Add DeFi position types (DeFiPosition, subtypes, enums) via Extended tier RFC
2. **contracts.md**: Add `getDeFiPositions()` to `BlockchainIntegrationContract`
3. **evm-integration**: Formalize DeFi submodule structure around existing Beefy+Aave adapters; add DeFiService facade; expose via `getDeFiPositions()`
4. **sol-integration**: Add DeFi submodule with Marinade, Raydium (LP), Jupiter, Orca adapters; add DeFiService; expose via `getDeFiPositions()`
5. **portfolio-aggregation** (po-e31): Implement DeFi aggregation — parallel fetch, dedup, pricing enrichment, portfolio inclusion
6. **asset-valuator**: Ensure pricing support for vault shares, LP tokens, staked derivatives (mSOL, stETH)
7. **cygnus-wealth-app**: Add DeFi positions view to dashboard

### Wave 5: SUI + Protocol Expansion

1. **sui-integration**: New bounded context for SUI chain data (addresses, tokens, transactions, DeFi)
2. **sui-integration DeFi**: Cetus, Turbos, Scallop adapters
3. **evm-integration**: Expand DeFi adapters (Uniswap, Compound, Lido, Curve, Convex)
4. **portfolio-aggregation**: No changes needed — SUI DeFi positions flow through automatically via the same `getDeFiPositions()` interface

### New Rigs/Repos Required

- **Wave 4**: None. All work fits in existing repos.
- **Wave 5**: `sui-integration` — new bounded context in Integration Domain. Repository: `cygnus-wealth/sui-integration`, package: `@cygnus-wealth/sui-integration`, topics: `domain-integration`, `context-sui`, `type-library`, `blockchain`, `web3`.

---

## Risks and Mitigations

| Risk | Mitigation |
|------|-----------|
| Chain integration libs grow too large with DeFi adapters | DeFi code is isolated in `src/defi/` submodule with its own adapter registry. If a chain eventually has 20+ protocol adapters, the DeFi submodule could be extracted to a separate package that the integration lib re-exports. This is a future optimization, not a present concern. |
| DeFi protocol ABIs/IDLs are large and chain-specific | Each adapter manages its own ABI/IDL. Shared utilities (multicall for EVM, getProgramAccounts batching for Solana) live within the chain lib. |
| Protocol adapter implementations diverge in quality | Define a `DeFiProtocolAdapter` interface within each chain lib. Each adapter implements the same interface. This is internal to the integration lib, not a cross-domain contract. |
| SUI integration delays block po-e31 | po-e31 is explicitly designed to work with available integrations only. It already handles partial failure (sol-integration down = EVM still works). SUI not existing is just another absent source. |

---

## Summary

| Question | Answer |
|----------|--------|
| Should DeFi live in chain libs or a separate lib? | **Chain libs (Option A).** DeFi reading is blockchain data acquisition. The cross-chain abstraction is a data-models concern. The cross-chain orchestration is a portfolio-aggregation concern. |
| Should po-e31 be multi-chain from the start? | **Yes.** po-e31 consumes `DeFiPosition[]` from a chain-agnostic interface. Any integration that implements `getDeFiPositions()` automatically participates. |
| What about Solana DeFi? | **Wave 4 scope: Marinade, Raydium, Jupiter, Orca.** Added to sol-integration as protocol adapters in a DeFi submodule. |
| What about SUI DeFi? | **Wave 5.** Requires new `sui-integration` bounded context first. Cetus, Turbos, Scallop are target protocols. |
| New repos needed? | **None for Wave 4.** One new repo (`sui-integration`) for Wave 5. |
| Dependency direction changes? | **None.** Existing dependency graph is preserved exactly. |
