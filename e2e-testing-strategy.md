# Enterprise E2E Testing Strategy

> **Navigation**: [Enterprise Architecture](./README.md) > E2E Testing Strategy

## Overview

This document defines the enterprise-wide end-to-end (E2E) testing strategy for CygnusWealth. It establishes which bounded contexts require E2E tests, what scenarios each must cover, what infrastructure is needed, and how E2E tests interact across domain boundaries.

E2E tests validate that the system works correctly from the user's perspective. They are the most expensive tests to write and maintain, so they must be deployed strategically where they provide the highest value.

### Relationship to Existing Testing Standards

The [Integration Domain Testing & Security Strategy](./domains/integration/testing-security.md) defines the testing pyramid for integration bounded contexts (80% unit, 15% integration, 4% contract, 1% E2E). This enterprise document extends that principle across all domains and provides the specific E2E policy that each rig architect must implement.

### Guiding Principles

1. **E2E tests verify user journeys, not implementation details** — test what the user sees and does, not internal plumbing
2. **Minimize E2E surface area** — only test critical paths that cannot be adequately covered by unit, integration, or contract tests
3. **Deterministic over comprehensive** — a small number of reliable E2E tests beats a large flaky suite
4. **Mock external boundaries, not internal ones** — mock RPC providers and APIs at the network edge, but let internal domain logic run for real
5. **Client-side sovereignty applies to tests too** — E2E tests must run without any backend server or centralized infrastructure beyond RPC endpoints

---

## Current State Assessment

| Bounded Context | E2E Tests? | Framework | Status | Network Calls |
|---|---|---|---|---|
| DataModels | No | Vitest 3.2.4 | N/A | No |
| EvmIntegration | Yes | Vitest 3.2.4 | Failing | Yes (Sepolia) |
| WalletIntegrationSystem | Yes | Vitest + Puppeteer | Functional | Mocked |
| AssetValuator | No | Jest 30.0.5 | N/A | No |
| SolIntegration | Yes | Vitest 3.2.4 | Functional | Yes (Testnet) |
| RobinhoodIntegration | No | Vitest 1.6.0 | N/A | No |
| PortfolioAggregation | No | Vitest 3.2.4 | N/A | No |
| CygnusWealthApp | Yes | Playwright 1.54.2 | Functional | Mocked |

### Key Gaps

- **EvmIntegration** has E2E tests but they are currently failing
- **AssetValuator** uses Jest instead of Vitest (framework inconsistency)
- **RobinhoodIntegration** uses an outdated Vitest version (1.6.0 vs 3.2.4)
- **PortfolioAggregation** has no E2E tests despite being the central orchestration layer
- No cross-boundary E2E tests exist — each repo tests in isolation

---

## Enterprise E2E Policy by Bounded Context

### 1. DataModels (Contract Domain)

**E2E tests appropriate: NO**

DataModels is a pure types package with no runtime behavior beyond TypeScript enum generation. It has no external dependencies, no network calls, and no side effects. Its correctness is fully verified by:
- Unit tests (type validation, enum coverage)
- Property-based tests (data structure invariants)
- Contract tests (schema compliance at consumption boundaries)

**Required instead:** Maintain 95% unit test coverage threshold. Contract tests in consuming bounded contexts validate DataModels compliance at the boundary.

---

### 2. EvmIntegration (Integration Domain)

**E2E tests appropriate: YES**

EvmIntegration connects to 8 live EVM blockchain networks. The critical risk is that RPC provider responses change format, rate limits shift, or chain configurations diverge from expectations. Unit tests with mocked responses cannot catch these regressions.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| Chain registry loads all supported chains | P0 | 15s | Verify all 8 EVM chains register correctly in production and testnet modes |
| Testnet RPC connectivity | P0 | 15s | Connect to Sepolia, verify health check returns valid block number |
| Native balance fetch | P0 | 15s | Fetch ETH balance from a known testnet address, verify correct wei-to-ether conversion |
| ERC-20 token balance fetch | P1 | 20s | Fetch token balances for a known testnet address with ERC-20 holdings |
| Multi-chain sequential fetch | P1 | 30s | Fetch balances across 3 chains sequentially, verify correct data transformation per chain |
| RPC failover | P1 | 30s | Configure intentionally invalid primary endpoint, verify fallback to secondary provider |
| Cache behavior | P2 | 15s | Verify cached responses return faster than network requests |

#### Infrastructure Requirements

- **Vitest E2E config:** Separate `vitest.e2e.config.ts` with node environment, 30s default timeout, sequential execution (single fork) to avoid RPC rate limiting, verbose reporter, 2 retries
- **Test wallets:** Well-known testnet addresses with stable balances (document addresses in test fixtures)
- **Network:** Sepolia testnet for Ethereum; other chain testnets as available
- **CI:** Run E2E tests on a separate schedule (not on every commit) — nightly or on merge to main

#### Current Action Required

Fix the failing `test:e2e` script. The `ChainRegistry.e2e.test.ts` test should align with the scenarios above.

---

### 3. WalletIntegrationSystem (Integration Domain)

**E2E tests appropriate: YES**

WalletIntegrationSystem manages the wallet connection lifecycle — detecting providers, connecting, switching chains, and fetching balances through the wallet provider API. These operations depend on the browser's `window.ethereum` interface which cannot be adequately tested with unit tests alone.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| MetaMask detection and connection | P0 | 10s | Inject mock ethereum provider, connect wallet, verify address returned |
| Chain switching | P0 | 10s | Connect to Ethereum, switch to Polygon, verify chain ID updates |
| Multi-chain balance fetch | P1 | 15s | Connect and fetch balances across Ethereum, Polygon, and BSC |
| Wallet disconnect and reconnect | P1 | 10s | Connect, disconnect, verify state cleared, reconnect successfully |
| Provider not available | P1 | 5s | No ethereum provider injected, verify graceful error handling |
| Multiple provider support | P2 | 15s | Inject multiple wallet providers, verify correct detection and listing |

#### Infrastructure Requirements

- **Mock ethereum provider:** Standardized mock implementing `window.ethereum` with configurable responses for `eth_requestAccounts`, `eth_chainId`, `eth_getBalance`, `wallet_switchEthereumChain`, and `eth_call`
- **Browser environment:** Puppeteer (current) or Playwright — either is acceptable, but Playwright is recommended for consistency with CygnusWealthApp
- **No network calls:** All E2E tests use mocked providers; no real blockchain access
- **Vitest E2E config:** jsdom environment for mock-based tests, Puppeteer/Playwright for browser injection tests

#### Standardization Note

The current dual approach (Puppeteer browser tests + Node-based mock tests) is acceptable. However, if the rig migrates to Playwright for browser tests, it should align with CygnusWealthApp's `addInitScript` pattern for provider injection.

---

### 4. AssetValuator (Portfolio Domain)

**E2E tests appropriate: YES (limited)**

AssetValuator fetches live pricing data from external APIs. The critical risk is API response format changes, rate limit behavior, and stale price detection. However, because pricing APIs are volatile and test environments may not mirror production, E2E tests should be minimal and focused on API connectivity rather than data accuracy.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| Price fetch for known asset | P0 | 10s | Fetch ETH/USD price, verify response contains valid numeric price > 0 |
| Batch price fetch | P1 | 15s | Fetch prices for ETH, BTC, SOL simultaneously, verify all return valid data |
| Currency conversion | P1 | 10s | Fetch price in USD and EUR, verify both return valid results |
| API unavailable fallback | P1 | 15s | Configure invalid API endpoint, verify fallback or cached value returned |
| Stale price detection | P2 | 10s | Verify price timestamp is within acceptable freshness window |

#### Infrastructure Requirements

- **Framework alignment:** Migrate from Jest to Vitest 3.2.4 to match enterprise standard — this is a prerequisite before adding E2E tests
- **Vitest E2E config:** Separate `vitest.e2e.config.ts`, node environment, 15s default timeout, 2 retries
- **API mocking:** For CI, use recorded API responses (recording/playback pattern); for nightly runs, allow live API calls
- **No API keys in tests:** Use free/public pricing endpoints or environment-variable-injected keys

#### Framework Migration Requirement

AssetValuator must migrate from Jest 30.0.5 to Vitest 3.2.4 before E2E tests are added. This is an enterprise consistency requirement. All bounded contexts must use Vitest as the test runner.

---

### 5. SolIntegration (Integration Domain)

**E2E tests appropriate: YES**

SolIntegration connects to Solana RPC endpoints for SOL balances, SPL tokens, and NFTs. It currently has the most mature E2E setup among the library repos. The existing strategy is sound and should be maintained with minor refinements.

#### E2E Scenarios (Current — Validated)

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| SOL balance fetch | P0 | 30s | Fetch SOL balance for known devnet wallet |
| Empty wallet handling | P0 | 30s | Verify zero balance returned for empty wallet |
| Invalid address handling | P0 | 30s | Verify graceful error for invalid Solana address |
| SPL token balance fetch | P1 | 30s | Fetch SPL tokens for wallet with known holdings |
| Full portfolio snapshot | P1 | 60s | Fetch complete portfolio and verify structure matches DataModels contracts |
| NFT detection | P1 | 30s | Fetch NFTs and verify mint/name/symbol/uri structure |
| RPC failover | P1 | 30s | Invalid primary endpoint triggers failover to secondary |
| Connection health reporting | P1 | 30s | Verify health metrics returned correctly |
| Cache invalidation | P2 | 30s | Clear cache, verify next request hits network |
| Concurrent request handling | P2 | 30s | 3 parallel balance requests complete within acceptable time |

#### Infrastructure (Current — No Changes Needed)

- Dedicated `vitest.e2e.config.ts` with 60s timeout, sequential execution (single fork), 2 retries, verbose reporter
- Devnet/testnet wallets with known balances
- Circuit breaker and metrics enabled during tests

#### Note

SolIntegration's E2E setup is the reference implementation for network-calling E2E tests. Other integration bounded contexts should follow its patterns for timeout management, sequential execution, and retry configuration.

---

### 6. RobinhoodIntegration (Integration Domain)

**E2E tests appropriate: YES (limited)**

RobinhoodIntegration communicates with the Robinhood API which requires OAuth authentication. Full E2E tests against the live API are impractical for automated CI. However, E2E tests should validate the integration pipeline using recorded API responses.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| Portfolio fetch with recorded response | P0 | 10s | Replay recorded Robinhood API response through full transformation pipeline, verify output matches DataModels contract |
| Holdings transformation | P1 | 10s | Recorded response with stocks, ETFs, and options — verify correct asset categorization |
| API authentication failure handling | P1 | 5s | Simulate 401 response, verify graceful error with user-facing message |
| Rate limit handling | P2 | 10s | Simulate 429 response, verify backoff behavior |
| Partial data response | P2 | 10s | Simulate response with missing optional fields, verify graceful degradation |

#### Infrastructure Requirements

- **Version upgrade:** Migrate from Vitest 1.6.0 to 3.2.4 — this is a prerequisite
- **Recording/playback:** Use a recorded API response fixture approach (JSON files with captured responses)
- **Vitest E2E config:** Separate `vitest.e2e.config.ts`, node environment, 15s default timeout
- **No live API calls in CI:** All E2E tests run against recorded fixtures
- **Manual E2E:** Document a procedure for manual E2E testing against the live Robinhood API with a test account

---

### 7. PortfolioAggregation (Portfolio Domain)

**E2E tests appropriate: YES**

PortfolioAggregation is the central orchestration layer. It coordinates data from all integration bounded contexts, reconciles assets, and produces the unified portfolio view. This is the highest-value E2E target after CygnusWealthApp because it validates that the integration contracts actually work together.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| Full portfolio sync with mocked integrations | P0 | 15s | Mock all integration endpoints (EVM, Solana, Robinhood), trigger portfolio sync, verify aggregated portfolio matches expected structure |
| Asset reconciliation across sources | P0 | 10s | Same asset (e.g., ETH) reported by EVM and wallet integrations — verify deduplication and correct total |
| Single integration failure isolation | P1 | 10s | One integration mock returns error, verify other integrations still contribute to portfolio |
| All integrations fail gracefully | P1 | 10s | All mocks fail, verify cached portfolio returned or empty state with appropriate error |
| Price enrichment via AssetValuator | P1 | 10s | Mock price responses, verify portfolio assets have correct USD valuations |
| Portfolio refresh lifecycle | P2 | 15s | Trigger refresh, verify loading state, verify updated data, verify completion state |
| Circuit breaker activation | P2 | 15s | Repeated failures trigger circuit breaker, verify fast-fail behavior |

#### Infrastructure Requirements

- **Vitest E2E config:** Separate `vitest.e2e.config.ts`, jsdom environment (React components involved), 20s default timeout
- **Integration mocks:** Mock implementations of each integration contract interface (EVM, Solana, Robinhood, AssetValuator) — these mock the contract boundary, not the internal implementation
- **Mock data fixtures:** Standardized fixture data from DataModels test fixtures (the DataModels repo already provides comprehensive fixtures in `tests/fixtures/`)
- **No network calls:** All integration boundaries are mocked
- **DDD layer alignment:** Tests should be organized under `src/tests/e2e/` following the existing DDD test organization pattern

---

### 8. CygnusWealthApp (Experience Domain)

**E2E tests appropriate: YES (primary E2E target)**

CygnusWealthApp is the user-facing application and the primary target for E2E testing. It validates the complete user experience from wallet connection through portfolio display. This is where E2E tests provide the highest value because they verify the entire stack from the user's perspective.

#### E2E Scenarios

| Scenario | Priority | Timeout | Description |
|---|---|---|---|
| **Wallet Connection Flow** | | | |
| Detect wallet and connect | P0 | 10s | Mock MetaMask provider, click Connect Wallet, verify address displayed |
| View connected wallet details | P0 | 10s | Navigate to wallet details after connection, verify address and chain info |
| Disconnect wallet | P0 | 10s | Disconnect wallet, verify state cleared and UI returns to disconnected state |
| **Portfolio Display** | | | |
| Display portfolio value | P0 | 10s | Mock portfolio data via Zustand store, verify total value rendered |
| Asset listing with balances | P1 | 10s | Verify asset table shows symbol, balance, and USD value columns |
| Zero balance filtering | P1 | 10s | Toggle zero-balance filter, verify assets appear/disappear correctly |
| **Environment Awareness** | | | |
| Testnet banner display | P0 | 10s | Configure testnet environment, verify testnet banner visible |
| Environment indicator badge | P1 | 10s | Verify environment indicator shows correct network |
| **Error Handling** | | | |
| Wallet connection failure | P1 | 10s | Mock provider that rejects connection, verify error message displayed |
| Portfolio load failure | P1 | 10s | Mock empty/error portfolio state, verify appropriate empty state UI |
| **Privacy** | | | |
| No external network calls | P0 | 10s | Verify no requests to analytics, telemetry, or tracking endpoints |

#### Infrastructure (Current — Validated)

- **Playwright** 1.54.2 with dedicated `playwright.config.ts`
- Projects: Chromium and Firefox
- Mock ethereum provider via `addInitScript`
- Zustand store state injection for portfolio data
- Vite dev server on port 5173 with `VITE_NETWORK_ENV=testnet`
- CI configuration: sequential workers, 2 retries, HTML reporter
- Screenshots on failure, traces on first retry

#### Note

CygnusWealthApp's Playwright setup is the reference implementation for browser-based E2E tests. Its mock provider injection pattern should be adopted by WalletIntegrationSystem if it migrates from Puppeteer.

---

## Cross-Boundary E2E Testing

### The Problem

Each bounded context currently tests in isolation. No tests validate that the contracts between domains actually work when the real implementations are wired together.

### Strategy: Layered Cross-Boundary Validation

Cross-boundary testing is addressed at three levels, not through a single monolithic E2E suite:

#### Level 1: Contract Tests (Per Bounded Context)

Each bounded context that consumes a contract must have contract tests that validate its outputs conform to the DataModels schema. This is already defined in the integration domain's testing strategy at 4% of the test mix.

**Owner:** Each rig architect
**Location:** Within each bounded context repo

#### Level 2: Integration Seam Tests (PortfolioAggregation)

PortfolioAggregation is the natural seam for cross-boundary validation because it consumes all integration contracts. Its E2E tests with mocked integrations validate that the contract interfaces are coherent and the orchestration logic works across boundaries.

**Owner:** PortfolioAggregation rig architect
**Location:** PortfolioAggregation repo, `src/tests/e2e/`

#### Level 3: User Journey Tests (CygnusWealthApp)

CygnusWealthApp E2E tests validate the full user journey through the real application with mocked external dependencies. This is the highest-level cross-boundary test because it exercises the Experience Domain's integration with Portfolio and Integration domains through the actual React component tree and state management.

**Owner:** CygnusWealthApp rig architect
**Location:** CygnusWealthApp repo, `e2e/`

### What NOT to Build

- **No enterprise-wide E2E test suite** — there is no monolithic test runner that wires all repos together. This would be fragile, slow, and violate domain isolation.
- **No shared test database or state** — each bounded context's tests are self-contained.
- **No live multi-repo integration tests** — real cross-boundary integration is validated by running CygnusWealthApp Playwright tests against a development build with all dependencies installed.

---

## Enterprise Test Infrastructure Standards

### Framework Requirements

| Standard | Requirement |
|---|---|
| Unit/Integration test runner | Vitest 3.x (all bounded contexts must use Vitest) |
| Browser E2E test runner | Playwright (for UI-based E2E tests) |
| Library E2E test runner | Vitest with separate `vitest.e2e.config.ts` |
| Coverage provider | @vitest/coverage-v8 |
| TypeScript execution | tsx (for standalone test scripts) |

### Vitest E2E Configuration Standard

All library bounded contexts with E2E tests must use a dedicated `vitest.e2e.config.ts` following this pattern:

- Environment: `node` (for library E2E) or `jsdom` (for React-dependent E2E)
- Test timeout: 30s default (60s for blockchain network calls)
- Hook timeout: 15s
- Include pattern: `src/test/e2e/**/*.test.ts`
- Reporter: `verbose`
- Retry: 2 (for network-dependent tests)
- Pool: `forks` with `singleFork: true` (sequential execution for rate-limit-sensitive tests)

### Playwright Configuration Standard

CygnusWealthApp (and any future UI bounded contexts) must use Playwright with:

- Test directory: `./e2e`
- Parallel execution: enabled locally, single worker in CI
- Retries: 2 in CI, 0 locally
- Reporter: HTML
- Trace: on first retry
- Screenshots: on failure only
- Browser projects: Chromium and Firefox minimum
- Dev server: auto-started via Vite

### Test Script Naming Convention

All bounded contexts must expose consistent npm script names:

| Script | Purpose |
|---|---|
| `test` | Run unit tests (default) |
| `test:e2e` | Run E2E tests (if applicable) |
| `test:coverage` | Run unit tests with coverage |

### CI/CD Execution Strategy

| Test Type | Trigger | Environment |
|---|---|---|
| Unit tests | Every push, every PR | No network access required |
| Contract tests | Every push, every PR | No network access required |
| E2E (mocked) | Every PR, merge to main | No network access required |
| E2E (network) | Nightly schedule, manual trigger | Testnet RPC access required |

Network-dependent E2E tests (EvmIntegration, SolIntegration) must not block PR merges. They run on a nightly schedule and failures trigger alerts, not build failures.

---

## Migration Roadmap

### Phase 1: Fix and Standardize (Immediate)

1. **Fix EvmIntegration E2E tests** — align with the scenarios defined above, ensure they pass on Sepolia
2. **Upgrade RobinhoodIntegration** to Vitest 3.2.4
3. **Migrate AssetValuator** from Jest to Vitest 3.2.4
4. **Standardize test:e2e scripts** across all repos that have E2E tests

### Phase 2: Fill Critical Gaps (Near-term)

1. **Add PortfolioAggregation E2E tests** — this is the highest-priority gap because it validates cross-boundary orchestration
2. **Add AssetValuator E2E tests** — after framework migration, add pricing API connectivity tests
3. **Add RobinhoodIntegration E2E tests** — recorded response fixtures through the transformation pipeline

### Phase 3: Mature and Monitor (Ongoing)

1. **Establish nightly E2E run** for network-dependent tests across EvmIntegration and SolIntegration
2. **Add E2E coverage metrics** to each bounded context's CI dashboard
3. **Review and prune E2E scenarios** quarterly — remove tests that duplicate coverage or are chronically flaky
4. **Expand CygnusWealthApp Playwright tests** as new features are added

---

## Trickle-Down Guidance

This enterprise strategy must be implemented by each rig architect within their bounded context. Each rig's ARCHITECTURE.md (or equivalent) should document:

1. Which E2E scenarios from this document apply to their context
2. The specific test infrastructure setup used
3. How to run E2E tests locally (`npm run test:e2e`)
4. Any rig-specific E2E considerations (test wallets, API fixtures, environment variables)

The enterprise architect will review rig-level E2E documentation during architecture reviews to ensure compliance with this strategy.
