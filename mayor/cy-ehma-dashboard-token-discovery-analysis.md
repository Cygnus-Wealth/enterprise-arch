# cy-ehma: CygnusWealthApp Dashboard Token Discovery Failure Analysis

## Issue Summary

Wallet `0x08ff8b099dcc50e212c998731b34a548826e1969` shows only **ETH on Ethereum** and **ETH on Arbitrum** in the CygnusWealthApp dashboard, despite holding:

- **Ethereum**: 0.082 ETH + 24 ERC-20 tokens (USDC, USDT, AICC, etc.) + 2 NFTs = ~$234
- **Arbitrum**: 0.005 ETH + 17 ERC-20 tokens (RAM, USDC.e, rETH, ALCX, TURBO, etc.) + 5 NFTs = ~$19
- **34 total chains** with assets, including Base (AERO, largest position at 26.80%), Polygon, BSC, Avalanche, Optimism, and others = ~$466 total

Console errors about "getting balances and values" are reported.

---

## Root Cause Analysis

### Finding 1: EvmIntegration — Broken ERC-20 Token Discovery (ROOT CAUSE)

**Severity: CRITICAL**
**Bounded Context: evm-integration**

The dashboard returns ONLY native ETH balances. This means EvmIntegration is successfully calling the standard RPC method for native balance (equivalent to `eth_getBalance`) but is **failing to discover and fetch ERC-20 token balances**.

**Why this is an architectural gap**: Standard EVM RPC calls (`eth_getBalance`) only return native token balances. ERC-20 token discovery requires one of:

1. **Enhanced/indexed APIs** (e.g., Alchemy's `alchemy_getTokenBalances`, `alchemy_getTokenMetadata`)
2. **Transfer event log scanning** (scan `Transfer(address,address,uint256)` events to/from the address)
3. **External token list registries** (query known token contract addresses per chain)

The EvmIntegration bounded context spec states it handles "Balance Queries: Fetch native and ERC-20 token balances" but **does not specify the token discovery mechanism** — how it determines WHICH ERC-20 contracts to query for a given address. This is a specification gap that has resulted in an implementation that likely only performs native balance queries.

**Corroborating evidence**: The enterprise E2E testing strategy (e2e-testing-strategy.md, line 31) explicitly states:

> **EvmIntegration** has E2E tests but they are **currently failing**

The E2E scenarios include "ERC-20 token balance fetch" as a P1 priority test — this test is failing, confirming the token discovery pathway is broken.

**Impact**: Only 2 of 41 tokens on Ethereum display. Only 1 of 18 tokens on Arbitrum display. Zero tokens from 6 other supported chains display. This renders the portfolio dashboard nearly useless.

---

### Finding 2: EvmIntegration — Incomplete Multi-Chain Activation (ROOT CAUSE)

**Severity: HIGH**
**Bounded Context: evm-integration**

CygnusWealth supports 8 EVM chains (Ethereum, Polygon, Arbitrum, Optimism, BNB Smart Chain, Avalanche, Base, Fantom). The wallet has confirmed balances on at least 6 of these 8 chains. Yet only Ethereum and Arbitrum return ANY data (native ETH only).

This suggests either:

- The tracking service is only tracking the address on 2 of 8 chains
- RPC connections for the other 6 chains are failing silently
- Chain registry initialization is incomplete (the E2E test "Chain registry loads all supported chains" is a P0 test that is likely failing)

The wallet's largest position ($125+ in AERO on Base, 26.80% of portfolio) is completely invisible.

---

### Finding 3: PortfolioAggregation — Silent Failure Masking (CONTRIBUTING)

**Severity: HIGH**
**Bounded Context: portfolio-aggregation**

Portfolio Aggregation's "partial results" strategy is architecturally correct for resilience, but creates a dangerous failure mode in this scenario. The spec states:

- "Continue aggregation even if individual sources fail"
- "Mark failed sources in response metadata"
- "Cache last known good data for failed sources"

When EVM Integration fails to discover ERC-20 tokens (or fails entirely for 6 of 8 chains), Portfolio Aggregation:

1. Accepts the partial result (native ETH only from 2 chains)
2. Marks failures in metadata (consumed only by logging, not surfaced to user)
3. Returns a "successful" portfolio with 2 assets

The console errors about "getting balances" originate here — Portfolio Aggregation catches exceptions from EVM Integration and logs them, but continues with whatever data succeeded. **The user sees a "complete" dashboard with no indication that 95%+ of their assets are missing.**

This is the architectural equivalent of a smoke alarm that logs the fire to a file but doesn't sound the alarm.

---

### Finding 4: AssetValuator — Cascading Price Lookup Failures (CONTRIBUTING)

**Severity: MEDIUM**
**Bounded Context: asset-valuator**

The console errors about "getting values" likely originate from Asset Valuator. Two possible failure modes:

**Mode A: Cascading from missing tokens.** If Portfolio Aggregation passes malformed or incomplete asset identifiers (e.g., tokens without contract addresses due to the discovery failure), Asset Valuator's `getBatchPrices` call would fail for those entries, generating console errors.

**Mode B: Independent pricing failures.** Even for the 2 assets that ARE discovered (ETH on Ethereum, ETH on Arbitrum), the pricing API calls (CoinGecko/CoinMarketCap) may be failing independently due to rate limits, API key issues, or incorrect symbol mapping. The E2E testing strategy notes Asset Valuator has **no E2E tests** and is still on Jest instead of Vitest (framework inconsistency).

If pricing fails and error handling drops the asset entirely (rather than showing "price unavailable"), this could further reduce visible assets. However, since 2 ETH entries ARE showing, Mode A is more likely for the majority of errors.

---

### Finding 5: CygnusWealthApp — No Degradation Visibility (MINOR)

**Severity: MEDIUM**
**Bounded Context: cygnus-wealth-app**

CygnusWealthApp is NOT the root cause — it correctly delegates to Portfolio Aggregation. However, it contributes to the problem by:

1. **Not displaying partial failure warnings**: When Portfolio Aggregation returns a "successful" result with failure metadata, CWA shows the data without warning the user that most sources failed
2. **No per-chain status indicators**: No way for the user to see "Ethereum: 1/24 tokens loaded, Polygon: failed, Base: failed"
3. **Error state swallowing**: CWA's error handling (line 158-168 in spec) says "Show partial data with warning" for integration failures, but this warning mechanism may not be triggered because Portfolio Aggregation reports success (with failure metadata buried in the response)

---

## Which Rigs Need Fixes

### Priority 1: evm-integration (CRITICAL — Root Cause)

**What to fix:**

1. **Implement explicit ERC-20 token discovery mechanism** — The architecture must specify how tokens are discovered. Recommended approach: use Alchemy Enhanced APIs (`alchemy_getTokenBalances`) as primary discovery method, with Transfer event log scanning as fallback. The RPC strategy already recommends Alchemy as the primary provider for all EVM chains except Fantom.

2. **Fix multi-chain activation** — Ensure the tracking service registers the address across ALL 8 supported chains, not just Ethereum and Arbitrum. The chain registry initialization must be validated (the P0 E2E test for this is currently failing).

3. **Fix E2E tests** — The e2e-testing-strategy.md already calls this out as "Phase 1: Immediate." The ERC-20 token balance fetch E2E test (P1) must pass on Sepolia with a test wallet that has known ERC-20 holdings.

4. **Add NFT discovery** — The spec mentions ERC-721/1155 support but this wallet's NFTs (2 on Ethereum, 5 on Arbitrum) are not appearing, suggesting NFT enumeration is also broken or unimplemented.

---

### Priority 2: portfolio-aggregation (HIGH — Failure Visibility)

**What to fix:**

1. **Surface failure severity to consumers** — The portfolio response must distinguish between "all sources succeeded" and "partial data due to failures." A simple `completeness` field (e.g., `{ complete: false, failedSources: ['evm:polygon', 'evm:base', ...], discoveredAssets: 2, expectedMinimum: 'unknown' }`) would let CWA decide how to alert the user.

2. **Add integration health reporting** — Expose a `getIntegrationHealth()` query that CWA can call to display per-chain status. This already aligns with the existing Observer pattern (Source Sync Completed Event, Error Occurred Event) but those events need to be structured for UI consumption.

3. **Add E2E tests** — The e2e-testing-strategy.md identifies Portfolio Aggregation as having NO E2E tests despite being "the central orchestration layer" and calls it "the highest-priority gap" in Phase 2. The P0 scenario "Full portfolio sync with mocked integrations" would have caught this issue pattern.

---

### Priority 3: asset-valuator (MEDIUM — Error Cascade)

**What to fix:**

1. **Migrate from Jest to Vitest** — Framework inconsistency is called out in e2e-testing-strategy.md as a prerequisite.

2. **Add resilient batch pricing** — `getBatchPrices` must not fail entirely when some symbols are unknown. Return prices for known assets with `null`/error for unknown ones. Never drop valid assets from the response due to pricing failures.

3. **Add E2E connectivity tests** — The P0 scenario "Price fetch for known asset" would validate basic API connectivity.

---

### Priority 4: cygnus-wealth-app (MEDIUM — User Experience)

**What to fix:**

1. **Display partial failure warnings** — When Portfolio Aggregation reports incomplete data, show a banner: "Some assets may not be displayed. X of Y chains responded successfully."

2. **Add per-chain status indicators** — Show connection/sync status for each configured chain so users can see which chains are healthy vs failed.

3. **Surface console errors as user-visible diagnostics** — Add a "diagnostics" panel (or at minimum, a health indicator) that surfaces integration health without requiring users to open browser devtools.

---

## Architectural Observations

### Critical Spec Gap: Token Discovery Mechanism

The EVM Integration bounded context specification describes WHAT it delivers (native + ERC-20 balances) but not HOW it discovers tokens. For native balances, the "how" is obvious (single RPC call). For ERC-20 tokens, the "how" is non-trivial and requires explicit architectural guidance:

- Standard RPC has no "list all tokens for address" method
- Alchemy Enhanced APIs provide this but are provider-specific
- Fallback mechanisms need specification (what if Enhanced API is unavailable?)
- Token metadata resolution needs specification (how to get symbol, decimals, name)

This spec gap should be addressed by the EVM Integration rig architect with Domain Architect approval.

### Partial Success Anti-Pattern

The system's layered "partial results" strategy (EvmIntegration → PortfolioAggregation → CygnusWealthApp) creates a failure mode where each layer accepts degraded data without raising alarms. The result is a dashboard that appears to work but shows 4% of actual assets. This is worse than a hard failure because the user may not realize data is missing.

**Recommendation**: Establish a "minimum viable portfolio" threshold. If fewer than N% of expected data sources respond successfully, escalate from "partial success" to "degraded mode" with explicit user notification.

### E2E Testing Gap Alignment

The existing E2E testing strategy document already identifies every gap relevant to this issue:
- EvmIntegration E2E tests failing (Phase 1 fix)
- PortfolioAggregation has no E2E tests (Phase 2, highest priority gap)
- AssetValuator on wrong test framework (Phase 1 prerequisite)

Executing the E2E migration roadmap would have prevented this issue from reaching users.

---

## Recommended Fix Order

| Order | Rig | Fix | Rationale |
|-------|-----|-----|-----------|
| 1 | evm-integration | Implement ERC-20 token discovery via Alchemy Enhanced APIs | Root cause — nothing else matters if tokens aren't discovered |
| 2 | evm-integration | Fix multi-chain activation (all 8 chains) | Second root cause — 6 of 8 chains returning no data at all |
| 3 | evm-integration | Fix failing E2E tests | Prevents regression, validates the above fixes |
| 4 | portfolio-aggregation | Add failure completeness reporting to portfolio response | Enables CWA to warn users about partial data |
| 5 | cygnus-wealth-app | Display partial failure warnings and per-chain status | User can see degradation instead of silently missing assets |
| 6 | asset-valuator | Resilient batch pricing (don't drop assets on price failure) | Prevents secondary asset loss |
| 7 | asset-valuator | Migrate Jest to Vitest, add E2E tests | Infrastructure standardization |
| 8 | portfolio-aggregation | Add E2E tests for orchestration scenarios | Validates cross-boundary integration |
