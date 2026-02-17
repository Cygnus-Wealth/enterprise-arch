# en-9qon: Cross-Chain DeFi Architecture Recommendation

**Status**: RECOMMENDATION (Updated: en-v99r dual-path discovery)
**Date**: 2026-02-17
**Bead**: en-9qon (amended by en-v99r)
**Scope**: DeFi integration structure, multi-chain DeFi scope, dual-path discovery, po-e31 guidance

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

---

## Dual-Path Discovery Architecture (en-v99r)

The original recommendation assumed DeFi discovery is a single activity: protocol adapters query contracts and return positions. In practice, DeFi positions are discovered through **two fundamentally different paths** that must work together. This section defines how both paths integrate, how deduplication works, and introduces the DeFi Token Registry.

### The Two Discovery Paths

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                    DeFiService (per chain integration)                        │
│                                                                              │
│  PATH 1: Passive Discovery (Wallet Token Scanning → Enrichment)             │
│  ┌──────────────┐    ┌──────────────────┐    ┌──────────────────────┐       │
│  │ Token Scanner │───▶│ DeFi Token       │───▶│ Protocol Adapter     │       │
│  │ (getTokens)   │    │ Registry         │    │ (enrich mode)        │       │
│  │               │    │                  │    │                      │       │
│  │ Finds ERC20s  │    │ "Is this token   │    │ Resolves underlying  │       │
│  │ in wallet     │    │  a DeFi receipt  │    │ assets, APY, value,  │       │
│  │               │    │  token?"         │    │ exchange rate        │       │
│  └──────────────┘    └──────────────────┘    └──────────────────────┘       │
│       Discovers:                                    Outputs:                 │
│       aTokens, mooTokens,                          DeFiPosition with        │
│       stETH, rETH, LP tokens,                      full protocol context    │
│       yTokens, cTokens                                                      │
│                                                                              │
│  PATH 2: Active Discovery (Contract State Querying)                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ Protocol Adapter (scan mode)                                      │       │
│  │                                                                    │       │
│  │ Queries protocol contracts directly for positions that             │       │
│  │ have NO wallet token representation:                              │       │
│  │  - MasterChef.userInfo(pid, address)  → farmed LP positions      │       │
│  │  - VestingContract.vestingSchedule()  → locked/vesting tokens    │       │
│  │  - PerpProtocol.getPosition()         → leveraged positions      │       │
│  │  - StakingPool.stakedBalance()        → staked tokens (no receipt)│       │
│  └──────────────────────────────────────────────────────────────────┘       │
│       Outputs: DeFiPosition[]                                                │
│                                                                              │
│  MERGE + DEDUPLICATE                                                         │
│  ┌──────────────────────────────────────────────────────────────────┐       │
│  │ DeFi Position Reconciler                                          │       │
│  │  - Combines Path 1 + Path 2 results                              │       │
│  │  - Deduplicates by (protocol, chain, pool_id, address)           │       │
│  │  - Removes receipt tokens from regular token list                │       │
│  │  - Returns unified DeFiPosition[]                                │       │
│  └──────────────────────────────────────────────────────────────────┘       │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Path 1: Wallet Token Scanning → DeFi Enrichment

**What it covers**: DeFi positions that appear as ERC20/ERC721 tokens in the user's wallet.

| Token Type | Examples | Protocol |
|-----------|----------|----------|
| Vault receipt tokens | mooTokens (Beefy), yTokens (Yearn) | Yield vaults |
| Lending supply tokens | aTokens (Aave), cTokens (Compound) | Lending |
| Liquid staking derivatives | stETH (Lido), rETH (Rocket Pool), mSOL (Marinade) | Staking |
| LP tokens (fungible) | Uniswap V2 LP, SushiSwap LP, Raydium LP | Liquidity |
| LP tokens (NFT) | Uniswap V3 NFT positions, Orca Whirlpool NFTs | Concentrated liquidity |

**The problem**: The existing token scanner (cy-ehma's ERC20 discovery via Alchemy Enhanced APIs or transfer log scanning) already finds these tokens in the wallet. But it treats them as regular ERC20 tokens — showing "mooBeefyETH-WETH" as an unknown token with no USD value. The token scanner has no protocol awareness.

**The solution**: After the token scanner returns its results, DeFiService runs each discovered token through the DeFi Token Registry. Matched tokens are pulled out of the regular token list and routed to the appropriate protocol adapter for enrichment — resolving the underlying assets, current exchange rate, APY, and USD value.

**Flow within DeFiService**:

1. Receive the full token list from the existing BalanceService/TokenScanner
2. For each token, check the DeFi Token Registry: `registry.identify(tokenAddress, chainId)`
3. If identified → remove from regular token list, group by protocol
4. For each protocol group → call the protocol adapter in **enrich mode** with the receipt token addresses and balances
5. Adapter returns `DeFiPosition[]` with full context (underlying assets, value, APY, position type)
6. Unidentified tokens remain in the regular token list as standard ERC20s

**Why Path 1 matters**: Many users' largest DeFi positions ARE receipt tokens sitting in their wallet. A user with $50,000 in Aave aUSDC currently sees an "unknown token" worth $0. Path 1 turns that into a recognized lending position worth $50,000 with a visible APY.

### Path 2: Contract State Querying → Active Discovery

**What it covers**: DeFi positions that leave the user's wallet entirely and exist only as state within protocol contracts.

| Position Type | Discovery Method | Why No Wallet Token |
|--------------|-----------------|---------------------|
| Farmed/staked LP tokens | MasterChef.userInfo(pid, addr) | LP token deposited into farm contract |
| Gauge-staked tokens | Gauge.balanceOf(addr) | Token deposited into governance gauge |
| Locked/vesting positions | VestingContract.schedules(addr) | Tokens held by vesting contract |
| Leveraged/perp positions | PerpProtocol.getPosition(addr) | Pure contract state, no receipt token |
| Borrow positions | LendingPool.getUserAccountData(addr) | Debt obligation, not a token |
| Some staking (no receipt) | StakingPool.stakes(addr) | Older protocols that don't mint receipt tokens |

**Flow within DeFiService**:

1. For each registered protocol adapter → call **scan mode** with the user's addresses
2. Each adapter queries its protocol's contracts for the address
3. Adapter returns `DeFiPosition[]` for any discovered positions (empty array if none)
4. Adapters should use multicall/batching to minimize RPC calls

**Why Path 2 matters**: A user who deposited their Uniswap LP tokens into a SushiSwap farm has zero visible tokens in their wallet for that position. Without active contract querying, those staked LP tokens are completely invisible. Similarly, borrow positions (Aave debt, Compound borrows) only exist as contract state.

### Protocol Adapter Dual-Mode Interface

Each protocol adapter supports both discovery paths through a single interface:

```
DeFiProtocolAdapter {
  // Identity
  protocolId: DeFiProtocol
  supportedChains: ChainId[]

  // PATH 1: Enrich receipt tokens already found in wallet
  // Called when DeFi Token Registry identifies tokens belonging to this protocol
  enrichTokens(
    walletTokens: { address: TokenAddress, balance: BigNumber }[],
    ownerAddress: Address,
    chainId: ChainId
  ): DeFiPosition[]

  // PATH 2: Actively scan for positions not visible in wallet
  // Called for every registered adapter on every refresh
  scanPositions(
    ownerAddress: Address,
    chainId: ChainId
  ): DeFiPosition[]

  // Registry contribution: which receipt tokens does this protocol mint?
  getReceiptTokens(chainId: ChainId): ReceiptTokenMapping[]
}
```

Adapters that only have receipt tokens (e.g., Lido's stETH) may implement `scanPositions` as a no-op. Adapters that only have non-wallet positions (e.g., a pure farming protocol) may implement `enrichTokens` as a no-op. Most protocols need both: Aave has aTokens (Path 1) AND borrow positions (Path 2).

### DeFi Token Registry

The DeFi Token Registry is the component that connects Path 1 to the protocol adapters. It maps known receipt token addresses to their protocols, enabling the token scanner's output to be recognized as DeFi positions.

**Structure**:

```
DeFiTokenRegistry {
  // Core lookup: is this token address a known DeFi receipt token?
  identify(tokenAddress: Address, chainId: ChainId): ReceiptTokenInfo | null

  // Bulk lookup for efficiency (called once per token scan batch)
  identifyBatch(
    tokens: { address: Address, chainId: ChainId }[]
  ): Map<Address, ReceiptTokenInfo>

  // Registry population: called at initialization and periodically
  refresh(chainId: ChainId): void
}

ReceiptTokenInfo {
  protocolId: DeFiProtocol
  positionType: DeFiPositionType
  underlyingAsset: Address | null    // known statically for some tokens
  factoryAddress: Address | null     // for dynamically-deployed receipt tokens
}
```

**Registry data sources** (layered, from fastest to most comprehensive):

1. **Static mappings** — hardcoded addresses for well-known, stable receipt tokens:
   - stETH: `0xae7ab96520DE3A18E5e111B5EaAb095312D7fE84` → Lido, staking
   - rETH: `0xae78736Cd615f374D3085123A210448E74Fc6393` → Rocket Pool, staking
   - aUSDC (Aave V3, Ethereum): `0x98C23E9d8f34FEFb1B7BD6a91B7FF122F4e16F5c` → Aave, lending_supply
   - These are deployed once and never change. Static mapping is correct.

2. **Factory-pattern matching** — for protocols that deploy receipt tokens dynamically via factory contracts:
   - Beefy deploys mooTokens per vault via its vault factory. Query the factory's vault list and cache.
   - Uniswap V2 LP tokens are created by the factory at `0x5C69bEe701ef814a2B6a3EDD4B1652CB9cc5aA6f`. Query `allPairs()`.
   - Yearn vault tokens deployed per strategy. Query the vault registry.
   - The adapter's `getReceiptTokens()` method handles this — DeFiService calls it during registry refresh.

3. **On-chain heuristics** (fallback) — for tokens not in static or factory mappings:
   - Check if the token contract implements known interfaces (e.g., `asset()` for ERC-4626 vaults, `underlying()` for cTokens)
   - This is a slow path — only used for unrecognized tokens as a last resort
   - Results are cached once identified

**Registry location**: The DeFi Token Registry lives inside each chain integration's DeFi submodule. It is chain-specific — EVM receipt token addresses are meaningless on Solana. Each chain integration manages its own registry instance.

**Registry refresh strategy**:
- Static mappings loaded at initialization (zero RPC cost)
- Factory-pattern refresh on first load and then every 24 hours (or on manual refresh)
- On-chain heuristic checks only when encountering unrecognized tokens
- All results cached in the integration's existing cache infrastructure

### Deduplication Strategy

A single DeFi position can appear in BOTH paths. Example: a user holds an Aave aUSDC token in their wallet (Path 1) AND has an active borrow against it (Path 2). These are different positions and should NOT be deduplicated. But: a user who has LP tokens AND staked those same LP tokens would show the LP in Path 1 (wallet scan) and the staked LP in Path 2 (farm scan) — if the LP tokens left the wallet entirely for the farm, Path 1 finds nothing (correct). If only SOME were staked, Path 1 finds the unstaked remainder (correct, no overlap).

The real deduplication risk is within a single path:

**Within-path deduplication key**: `(protocol, chainId, positionType, poolOrMarketId, ownerAddress)`

| Scenario | Path 1 | Path 2 | Overlap? | Resolution |
|----------|--------|--------|----------|-----------|
| aUSDC in wallet | aUSDC balance → lending_supply position | Borrow position for same market | No | Different positionType (supply vs borrow). Both correct. |
| LP token in wallet, not staked | LP balance → liquidity_pool position | Farm scan finds nothing for this LP | No | Path 2 returns empty. No conflict. |
| LP token fully staked in farm | Wallet scan finds no LP token | Farm scan finds staked LP | No | Path 1 has nothing to enrich. Path 2 discovers it. |
| LP token partially staked | Wallet has remaining LP → liquidity_pool position | Farm scan finds staked portion → farming position | No | Different positionType (liquidity_pool vs farming). Both correct. |
| Same protocol queried by both paths | stETH in wallet → staking position | Lido adapter scanPositions also queries stETH balance | **Yes** | DeFi Position Reconciler deduplicates by key. Path 1 result wins (already enriched from actual wallet balance). |

**Reconciler rules**:
1. Compute dedup key for every position from both paths
2. If a key appears in both paths, prefer the Path 1 result (it's derived from the actual wallet token balance, which is ground truth)
3. If a key appears only in one path, include it
4. The reconciler also removes receipt tokens from the regular token list — any token identified by the registry as a DeFi receipt token must NOT appear in both `getTokens()` results AND `getDeFiPositions()` results

### Coordination with Existing Token Discovery (cy-ehma)

The cy-ehma analysis identified that ERC-20 token discovery via Alchemy Enhanced APIs (or transfer log scanning) is the mechanism for finding tokens in a wallet. This is Path 1's input.

**Current flow** (broken for DeFi):
```
Alchemy getTokenBalances → all ERC20s → TokenList
  Result: [ETH, USDC, aUSDC, mooBeefyETH, stETH, UNI-V2-LP, ...]
  Problem: aUSDC, mooBeefyETH, stETH, UNI-V2-LP show as unknown tokens
```

**Updated flow** (dual-path):
```
Alchemy getTokenBalances → all ERC20s
  ├─► DeFi Token Registry filter
  │     ├─► Identified DeFi tokens → Protocol adapters (enrich mode) → DeFiPosition[]
  │     └─► Regular tokens → TokenList (aUSDC, mooBeefyETH removed)
  │
  ├─► Protocol adapters (scan mode) → DeFiPosition[] (farm, borrow, vesting positions)
  │
  └─► DeFi Position Reconciler → deduplicated DeFiPosition[]
```

**Key coordination point**: `getTokens()` and `getDeFiPositions()` share the same underlying token scan data. The token scan runs once. DeFiService receives the raw scan results, splits them into regular tokens and DeFi receipt tokens, enriches the DeFi tokens, adds actively-scanned positions, deduplicates, and returns the final lists. This means `getTokens()` must be aware of the DeFi split — it should NOT return receipt tokens that have been claimed by DeFiService. The simplest implementation: DeFiService processes first, marks identified tokens, then TokenService/BalanceService filters them out.

### Solana Path Equivalents

The dual-path model applies to Solana with chain-specific mechanics:

**Path 1 (Passive — SPL Token Scanning)**:
- SPL token accounts are discovered via `getTokenAccountsByOwner`
- Receipt tokens: mSOL (Marinade), LP tokens (Raydium, Orca)
- Orca Whirlpool positions are NFTs (Metaplex) — discovered via NFT scan, identified by the registry
- DeFi Token Registry maps SPL token mints to protocols

**Path 2 (Active — Program Account Querying)**:
- Marinade: query stake account state for native staking (not liquid mSOL)
- Raydium: query farm program accounts for staked LP
- Jupiter: query DCA, limit order, and perp program accounts
- Orca: query Whirlpool program for tick data and fee accrual (enriches NFT positions from Path 1)

**Solana-specific difference**: Solana programs use Program Derived Addresses (PDAs) rather than mapping slots. The adapter's `scanPositions` derives PDAs from the user's address and queries the corresponding accounts. The DeFi Token Registry maps SPL token mint addresses to protocols (same concept as EVM, different address type).

### Updated Internal Structure

The DeFi submodule structure from the original recommendation is updated to include the registry and reconciler:

```
evm-integration/
├── src/
│   ├── services/
│   │   ├── BalanceService.ts
│   │   ├── TokenService.ts          # Returns tokens MINUS DeFi receipt tokens
│   │   ├── TransactionService.ts
│   │   ├── TrackingService.ts
│   │   └── DeFiService.ts          # Orchestrates dual-path discovery
│   ├── defi/
│   │   ├── index.ts
│   │   ├── types.ts                 # Adapter interface (enrichTokens + scanPositions)
│   │   ├── registry.ts              # DeFi Token Registry (per-chain)
│   │   ├── reconciler.ts            # Deduplication + token list coordination
│   │   ├── adapters/
│   │   │   ├── beefy-adapter.ts     # Path 1: mooToken enrichment. Path 2: no-op
│   │   │   ├── aave-adapter.ts      # Path 1: aToken enrichment. Path 2: borrow positions
│   │   │   ├── uniswap-adapter.ts   # Path 1: LP token enrichment. Path 2: no-op
│   │   │   ├── compound-adapter.ts  # Path 1: cToken enrichment. Path 2: borrow positions
│   │   │   ├── lido-adapter.ts      # Path 1: stETH enrichment. Path 2: no-op
│   │   │   └── sushiswap-adapter.ts # Path 1: SLP enrichment. Path 2: MasterChef staked positions
│   │   └── utils/
│   │       ├── abi-decoder.ts
│   │       └── multicall.ts
│   └── ...
```

```
sol-integration/
├── src/
│   ├── services/
│   │   ├── BalanceService.ts
│   │   ├── NFTService.ts            # Returns NFTs MINUS DeFi position NFTs
│   │   ├── TransactionService.ts
│   │   └── DeFiService.ts
│   ├── defi/
│   │   ├── index.ts
│   │   ├── types.ts
│   │   ├── registry.ts              # Maps SPL token mints to protocols
│   │   ├── reconciler.ts
│   │   ├── adapters/
│   │   │   ├── marinade-adapter.ts  # Path 1: mSOL enrichment. Path 2: native stake accounts
│   │   │   ├── raydium-adapter.ts   # Path 1: LP enrichment. Path 2: farmed LP positions
│   │   │   ├── jupiter-adapter.ts   # Path 1: no-op. Path 2: DCA, limit orders, perps
│   │   │   └── orca-adapter.ts      # Path 1: Whirlpool NFT enrichment. Path 2: fee accrual
│   │   └── utils/
│   │       └── account-parser.ts
│   └── ...
```

---

### Contract Changes Required in data-models

New types needed (Extended tier — expedited RFC):

- `DeFiPositionType` enum: `vault`, `lending_supply`, `lending_borrow`, `liquidity_pool`, `staking`, `farming`, `perp_position`, `vesting`
- `DeFiProtocol` enum: `beefy`, `aave`, `uniswap`, `compound`, `lido`, `rocket_pool`, `yearn`, `sushiswap`, `marinade`, `raydium`, `jupiter`, `orca`, `cetus`, `turbos`, `scallop`
- `DeFiDiscoveryPath` enum: `wallet_token_scan`, `contract_state_query` — records HOW a position was discovered, critical for debugging and deduplication
- `DeFiPosition` interface: core position with `type`, `protocol`, `chain`, `underlyingAssets[]`, `value`, `apy`, `rewards[]`, `discoveryPath`, `receiptToken?`, `deduplicationKey`, `metadata`
- `VaultPosition` extends `DeFiPosition`: auto-compound vault specifics, exchange rate to underlying
- `LendingPosition` extends `DeFiPosition`: supply/borrow rates, collateral factor, health factor
- `LiquidityPosition` extends `DeFiPosition`: token pair, price range (for concentrated liquidity), fee tier, impermanent loss
- `StakingPosition` extends `DeFiPosition`: lock period, validator/operator, rewards accrued
- `FarmingPosition` extends `DeFiPosition`: underlying LP token, farm contract, reward tokens, emission rate
- `ReceiptTokenMapping` type: `{ tokenAddress, chainId, protocolId, positionType, underlyingAsset?, factoryAddress? }` — used by the DeFi Token Registry (internal to each chain integration, not a cross-domain contract, but the structure is standardized in data-models for consistency)

These types define the cross-chain abstraction. Every chain-specific adapter transforms protocol-specific data into these types. The `discoveryPath` field enables po-e31 and debugging tools to understand whether a position was found via wallet token scanning or active contract querying. The `deduplicationKey` is a deterministic string computed as `{protocol}:{chainId}:{positionType}:{poolOrMarketId}:{ownerAddress}` — used by the reconciler to merge results from both discovery paths.

### Contract Change: BlockchainIntegrationContract Extension

The existing `BlockchainIntegrationContract` in contracts.md gains one method and one behavioral change:

```
BlockchainIntegrationContract {
  // Existing
  getBalances(addresses): BalanceList
  getTransactions(addresses, options): TransactionList
  getTokens(addresses): TokenList          // BEHAVIOR CHANGE: excludes DeFi receipt tokens
  getNFTs(addresses): NFTList              // BEHAVIOR CHANGE: excludes DeFi position NFTs (e.g., Uniswap V3)
  subscribeToUpdates(addresses): Subscription

  // NEW
  getDeFiPositions(addresses): DeFiPositionList
}
```

**Behavioral change for getTokens() and getNFTs()**: Once DeFiService is active, tokens identified by the DeFi Token Registry as receipt tokens are excluded from `getTokens()` results and included in `getDeFiPositions()` results instead. This prevents the same asset appearing twice — once as an "unknown token" and once as an enriched DeFi position. The same applies to DeFi-related NFTs (e.g., Uniswap V3 LP NFTs are returned by `getDeFiPositions()` as `LiquidityPosition`, not by `getNFTs()` as a generic NFT).

**Implementation**: `getDeFiPositions()` internally executes both discovery paths (wallet token scan enrichment + contract state querying), reconciles the results, and returns unified `DeFiPosition[]`. The token scan data is shared with `getTokens()` — both methods consume the same underlying scan, but DeFiService claims identified receipt tokens before TokenService returns its results.

### po-e31 Design Guidance (Multi-Chain from Start)

po-e31 in `portfolio-aggregation` should:

1. **Call `getDeFiPositions()` on all chain integrations in parallel** — same scatter-gather pattern already used for `getBalances()`. Each integration internally handles both discovery paths and returns a single unified `DeFiPosition[]`.
2. **Deduplicate cross-chain positions** — same protocol on multiple chains (e.g., Aave on Ethereum + Aave on Arbitrum) should be grouped but shown per-chain. The `deduplicationKey` on each position enables this.
3. **Aggregate DeFi value into portfolio totals** — sum DeFi position values into overall portfolio. Distinguish between supply/asset positions (add to total) and borrow/debt positions (subtract from total).
4. **Request pricing for underlying assets** via `asset-valuator` — LP tokens, vault shares need to be priced by their underlying composition. The `underlyingAssets[]` field on each position tells asset-valuator what needs pricing.
5. **Apply partial failure strategy** — if sol-integration DeFi fails, EVM DeFi positions are still returned with failure metadata. If Path 2 fails within an integration but Path 1 succeeds, partial DeFi results are still returned (receipt tokens are still enriched even if farm scanning fails).
6. **Expose DeFi positions as first-class portfolio content** alongside crypto holdings, TradFi positions, and NFTs
7. **Coordinate with token deduplication** — po-e31 must consume `getTokens()` and `getDeFiPositions()` together and verify no asset appears in both. If a receipt token escapes the DeFi filter (e.g., new protocol not yet in registry), po-e31 should not double-count it.

po-e31 does NOT need to know about Beefy, Aave, or Marinade specifically. It only sees `DeFiPosition[]` from each integration. The protocol-specific knowledge — including which discovery path found each position — is fully encapsulated in the chain integrations. The `discoveryPath` field is available for diagnostics but does not affect aggregation logic.

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

1. **data-models**: Add DeFi position types (DeFiPosition, subtypes, enums, DeFiDiscoveryPath, ReceiptTokenMapping) via Extended tier RFC
2. **contracts.md**: Add `getDeFiPositions()` to `BlockchainIntegrationContract`; document behavioral change to `getTokens()` and `getNFTs()` (receipt token exclusion)
3. **evm-integration**: Implement dual-path DeFi discovery:
   - DeFi Token Registry with static mappings + factory pattern for Beefy, Aave, Uniswap, Compound, Lido
   - DeFi Position Reconciler for deduplication and token list coordination
   - DeFiService facade orchestrating both paths
   - Protocol adapters with dual-mode interface (enrichTokens + scanPositions)
   - Coordinate with existing token scanner (cy-ehma) so receipt tokens are claimed by DeFiService
4. **sol-integration**: Implement dual-path DeFi discovery:
   - DeFi Token Registry mapping SPL token mints to protocols (mSOL, Raydium LP, Orca Whirlpool NFTs)
   - Protocol adapters for Marinade, Raydium, Jupiter, Orca with dual-mode interface
   - DeFiService and reconciler following same pattern as evm-integration
5. **portfolio-aggregation** (po-e31): Implement DeFi aggregation — parallel `getDeFiPositions()` fetch, cross-chain dedup, pricing enrichment, portfolio inclusion. Coordinate with `getTokens()` to prevent double-counting receipt tokens.
6. **asset-valuator**: Ensure pricing support for vault shares (exchange rate to underlying), LP tokens (underlying pair composition), staked derivatives (mSOL→SOL, stETH→ETH exchange rates)
7. **cygnus-wealth-app**: Add DeFi positions view to dashboard, including discovery path indicator for transparency

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
| Protocol adapter implementations diverge in quality | Define a `DeFiProtocolAdapter` interface within each chain lib (with `enrichTokens` + `scanPositions` dual-mode). Each adapter implements the same interface. This is internal to the integration lib, not a cross-domain contract. |
| SUI integration delays block po-e31 | po-e31 is explicitly designed to work with available integrations only. It already handles partial failure (sol-integration down = EVM still works). SUI not existing is just another absent source. |
| DeFi Token Registry becomes stale — new receipt tokens not recognized | Three-tier registry (static → factory → on-chain heuristic) ensures progressive discovery. Factory refresh catches new vaults/pools within 24 hours. On-chain heuristics catch ERC-4626 vaults even without explicit mapping. Unknown receipt tokens still appear as regular ERC20s (degraded but not lost). |
| Path 1 and Path 2 produce duplicate positions | Deterministic `deduplicationKey` and reconciler rules prevent double-counting. Path 1 wins on conflicts (wallet balance is ground truth). Comprehensive dedup scenario table in architecture doc covers all known cases. |
| Token scan and DeFi scan coupling creates ordering dependency | DeFiService orchestrates both — token scan runs first, DeFi registry filters second, active scan runs in parallel with enrichment. No circular dependency. TokenService waits for DeFi filtering before returning results. |
| Receipt token removal from getTokens() breaks existing consumers | Behavioral change is additive: tokens previously shown as "unknown $0" are now shown as enriched DeFi positions. No consumer loses data — they gain it. Consumers that relied on receipt token addresses in `getTokens()` should migrate to `getDeFiPositions()`. |
| Path 2 contract queries are expensive (many RPC calls per protocol) | Adapters use multicall (EVM) and batched getProgramAccounts (Solana) to minimize RPC calls. Path 2 scan runs with lower priority than Path 1 enrichment. Circuit breakers on individual adapters prevent a single broken protocol from degrading the entire DeFi scan. |

---

## Summary

| Question | Answer |
|----------|--------|
| Should DeFi live in chain libs or a separate lib? | **Chain libs (Option A).** DeFi reading is blockchain data acquisition. The cross-chain abstraction is a data-models concern. The cross-chain orchestration is a portfolio-aggregation concern. |
| Should po-e31 be multi-chain from the start? | **Yes.** po-e31 consumes `DeFiPosition[]` from a chain-agnostic interface. Any integration that implements `getDeFiPositions()` automatically participates. |
| How do the two DeFi discovery paths integrate? (en-v99r) | **Token scanner feeds DeFi Token Registry for Path 1 enrichment. Protocol adapters independently query contracts for Path 2.** DeFiService orchestrates both paths, and the reconciler deduplicates before returning unified positions. |
| How does token discovery (cy-ehma) coordinate with DeFi adapters? (en-v99r) | **Shared token scan data, split by registry.** The token scan runs once. DeFi Token Registry claims receipt tokens for enrichment. Remaining tokens go to `getTokens()`. No duplicate positions. |
| Should there be a DeFi token registry? (en-v99r) | **Yes, per-chain, inside each integration's DeFi submodule.** Three-tier: static mappings → factory-pattern discovery → on-chain heuristics. Maps receipt token addresses to protocols for Path 1 enrichment. |
| What about Solana DeFi? | **Wave 4 scope: Marinade, Raydium, Jupiter, Orca.** Added to sol-integration as protocol adapters in a DeFi submodule with dual-path discovery. |
| What about SUI DeFi? | **Wave 5.** Requires new `sui-integration` bounded context first. Cetus, Turbos, Scallop are target protocols. |
| New repos needed? | **None for Wave 4.** One new repo (`sui-integration`) for Wave 5. |
| Dependency direction changes? | **None.** Existing dependency graph is preserved exactly. |
