# en-p0ha: WebSocket-First Architecture for Real-Time Data

**Status**: RECOMMENDATION
**Date**: 2026-02-17
**Bead**: en-p0ha
**Scope**: Transport architecture, WebSocket subscriptions, polling deprecation, per-chain WS support, Zustand live update consumption

---

## Executive Summary

**The default transport for live data must be WebSocket. HTTP polling must be the fallback, not the primary path.**

The app currently polls RPC endpoints via HTTP for balance and token updates. This burns through free-tier quotas, produces stale data between poll intervals, and creates unnecessary load on rate-limited endpoints. The RPC strategy already notes "WebSocket connections preferred for real-time updates" for high-value chains, and both evm-integration and sol-integration specs list WebSocket support as a core responsibility — but no architectural directive exists defining how WebSocket subscriptions should be structured, managed, consumed, or degraded.

This directive establishes the WebSocket subscription architecture across all layers: Integration Domain (evm-integration, sol-integration), Portfolio Domain (portfolio-aggregation), and Experience Domain (CygnusWealthApp).

---

## 1. Operation Classification: WebSocket vs HTTP

### Principle

Operations that monitor **continuously changing state** use WebSocket subscriptions. Operations that retrieve **historical or static data** use HTTP requests. The boundary is temporal: "Is this data expected to change while the user is looking at it?"

### WebSocket Operations (Live Data)

| Operation | Domain | WS Method (EVM) | WS Method (Solana) | Rationale |
|-----------|--------|------------------|---------------------|-----------|
| Native balance changes | evm-integration / sol-integration | `eth_subscribe("newHeads")` + `eth_getBalance` on new block | `accountSubscribe(address)` | Balance changes with every block the user transacts in |
| ERC-20/SPL token balance changes | evm-integration / sol-integration | `eth_subscribe("logs", {topics: [Transfer]})` filtered to tracked addresses | `accountSubscribe(tokenAccount)` per tracked ATA | Token transfers are the primary portfolio event |
| Pending transaction status | evm-integration | `eth_subscribe("newPendingTransactions")` filtered to tracked addresses | `signatureSubscribe(txSig)` | Users need immediate feedback on pending txs |
| New block headers | evm-integration | `eth_subscribe("newHeads")` | `slotSubscribe()` | Drives balance re-checks and block confirmation counts |
| NFT transfer events | evm-integration / sol-integration | `eth_subscribe("logs", {topics: [Transfer], address: NFTContract})` | `programSubscribe(MetaplexProgram)` filtered to user | NFT arrivals/departures affect portfolio |
| DeFi position changes | evm-integration / sol-integration | `eth_subscribe("logs")` filtered to vault/pool contracts | `programSubscribe(protocolProgram)` | LP positions, staking rewards, vault shares change per block |

### HTTP Operations (Historical / Static / On-Demand)

| Operation | Domain | Method | Rationale |
|-----------|--------|--------|-----------|
| Transaction history retrieval | evm-integration / sol-integration | `eth_getBlockByNumber`, Alchemy Enhanced APIs, `getSignaturesForAddress` | Historical data is immutable once confirmed — no live updates needed |
| Token metadata (name, symbol, decimals) | evm-integration / sol-integration | `eth_call` to ERC-20 contract, Helius DAS API | Metadata is static, changes rarely if ever |
| Contract ABI resolution | evm-integration | Etherscan/Sourcify API or local cache | Static data, cached for days |
| NFT metadata (image, attributes) | evm-integration / sol-integration | IPFS/Arweave gateway, Helius DAS API | Static per token ID |
| Price data | asset-valuator | CoinGecko/CoinMarketCap REST API | Price APIs are HTTP-only; WS price feeds are a future enhancement |
| Portfolio refresh (full) | portfolio-aggregation | Aggregated HTTP batch across integrations | Initial load and manual refresh use HTTP; WS handles incremental updates afterward |
| Address discovery (Enhanced APIs) | evm-integration | Alchemy `alchemy_getTokenBalances` | Discovery is a one-time scan, not a subscription |
| Historical DeFi positions | portfolio-aggregation | HTTP queries to protocol subgraphs/APIs | Historical snapshots are immutable |

### Hybrid Operations

| Operation | Primary Transport | Fallback | Notes |
|-----------|-------------------|----------|-------|
| Balance fetch on dashboard load | HTTP (initial), then WS (updates) | HTTP polling at 30s interval | First paint uses HTTP for speed; WS takes over for live updates |
| Portfolio aggregation refresh | HTTP (full refresh), WS (incremental) | HTTP polling at 60s interval | Manual refresh button always triggers HTTP; live updates via WS |

---

## 2. WebSocket Connection Management

### Connection Lifecycle

Each integration maintains one WebSocket connection per chain per RPC provider tier. Connections are shared across all subscriptions for that chain.

```
┌──────────────────────────────────────────────────────┐
│                WebSocket Connection Pool               │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  │
│  │  Ethereum    │  │  Polygon    │  │  Arbitrum   │  │
│  │  WS conn    │  │  WS conn    │  │  WS conn    │  │
│  │  (Alchemy)  │  │  (Alchemy)  │  │  (Alchemy)  │  │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘  │
│         │                │                │          │
│    ┌────┴────┐      ┌────┴────┐      ┌────┴────┐    │
│    │ sub: hd │      │ sub: hd │      │ sub: hd │    │
│    │ sub: log│      │ sub: log│      │ sub: log│    │
│    │ sub: pnd│      │ sub: log│      └─────────┘    │
│    └─────────┘      └─────────┘                      │
│                                                        │
│  ┌─────────────┐  ┌─────────────┐                    │
│  │  Solana     │  │  Base       │                    │
│  │  WS conn    │  │  WS conn    │  ... (per chain)  │
│  │  (Helius)   │  │  (Alchemy)  │                    │
│  └──────┬──────┘  └──────┬──────┘                    │
│         │                │                            │
│    ┌────┴────┐      ┌────┴────┐                      │
│    │ sub: acc│      │ sub: hd │                      │
│    │ sub: prg│      │ sub: log│                      │
│    │ sub: slt│      └─────────┘                      │
│    └─────────┘                                        │
└──────────────────────────────────────────────────────┘
```

### Connection States

```
DISCONNECTED → CONNECTING → CONNECTED → SUBSCRIBING → ACTIVE
     ↑              │            │            │           │
     │              ↓            ↓            ↓           │
     └──── RECONNECTING ←───────────────────────────────┘
                │
                ↓ (max retries exceeded)
           FAILED → POLLING_FALLBACK
```

### Reconnection Strategy

The reconnection protocol uses exponential backoff aligned with the existing retry pattern from `patterns.md`:

- **Base delay**: 1000ms
- **Max delay**: 30000ms
- **Multiplier**: 2
- **Jitter factor**: 0.3
- **Max reconnect attempts**: 10 (before declaring connection failed and falling back to polling)
- **Reconnect formula**: `delay = min(1000 * (2 ^ attempt) + random(0, 300), 30000)`

On reconnection success:
1. Re-subscribe all active subscriptions for that chain
2. Perform a one-time HTTP fetch to fill any gap during disconnect
3. Emit `ConnectionRestored` domain event
4. Resume normal subscription flow

### Heartbeat / Keep-Alive

- **Ping interval**: 30 seconds (aligned with existing KeepAliveManager pattern)
- **Pong timeout**: 5 seconds — if no pong received, mark connection unhealthy
- **Missed pong threshold**: 2 consecutive — trigger reconnection
- EVM providers: Use WebSocket native ping/pong frames
- Solana (Helius/native): Use application-level ping (send lightweight `getHealth` or rely on subscription message flow)

### Multiplexing

All subscriptions for a given chain share a single WebSocket connection. This is the natural model for both EVM (`eth_subscribe` returns a subscription ID on the same connection) and Solana (`accountSubscribe` / `programSubscribe` return subscription IDs on the same connection).

**Subscription multiplexing per connection**:
- EVM: Up to 100 concurrent `eth_subscribe` subscriptions per connection (Alchemy limit; dRPC similar)
- Solana: Up to 40,000 concurrent subscriptions per Helius WebSocket connection (Helius limit)
- If subscription count exceeds provider limits, open a second connection to the same chain on the secondary provider

**Subscription registry**: Each connection maintains a `Map<subscriptionId, SubscriptionMetadata>` for:
- Resubscription on reconnect
- Cleanup on address removal
- Metrics tracking (messages received, last message timestamp)

### Lazy Connection Establishment

WebSocket connections are established lazily — only when the first subscription for a chain is requested. This prevents opening connections to chains where the user has no tracked addresses.

**Connection teardown**: When the last subscription for a chain is removed (user removes all addresses for that chain), the WebSocket connection is closed after a 60-second grace period (in case the user re-adds an address).

---

## 3. Integration Subscription Interfaces

### EVM Integration: Subscription Service

evm-integration exposes a `SubscriptionService` alongside its existing `BalanceService`, `TransactionService`, and `TrackingService`.

**SubscriptionService Interface**:

```
SubscriptionService {
  // Subscribe to balance changes for tracked addresses on a specific chain
  subscribeBalances(chainId: ChainId, addresses: Address[]): SubscriptionHandle

  // Subscribe to ERC-20 token transfer events for tracked addresses
  subscribeTokenTransfers(chainId: ChainId, addresses: Address[]): SubscriptionHandle

  // Subscribe to new block headers (used internally and by portfolio-aggregation)
  subscribeNewBlocks(chainId: ChainId): SubscriptionHandle

  // Subscribe to pending transactions for tracked addresses
  subscribePendingTransactions(chainId: ChainId, addresses: Address[]): SubscriptionHandle

  // Subscribe to specific contract events (for DeFi position tracking)
  subscribeContractEvents(chainId: ChainId, contractAddress: Address, topics: Topic[]): SubscriptionHandle

  // Unsubscribe and clean up
  unsubscribe(handle: SubscriptionHandle): void

  // Get subscription health status
  getSubscriptionStatus(chainId: ChainId): SubscriptionStatus
}
```

**SubscriptionHandle**:

```
SubscriptionHandle {
  id: string                        // Unique subscription ID
  chainId: ChainId                  // Which chain this subscription is on
  type: SubscriptionType            // 'balance' | 'tokenTransfer' | 'newBlock' | 'pendingTx' | 'contractEvent'
  status: 'active' | 'reconnecting' | 'failed'
  onData: (callback: (event: SubscriptionEvent) => void) => void
  onError: (callback: (error: SubscriptionError) => void) => void
  onStatusChange: (callback: (status: SubscriptionStatus) => void) => void
  unsubscribe: () => void
}
```

**Underlying EVM WebSocket Methods Used**:

| Subscription Type | EVM WS Method | Filter Parameters |
|-------------------|---------------|-------------------|
| Balance changes | `eth_subscribe("newHeads")` → triggers `eth_getBalance` per tracked address on each new block | Addresses from tracking service |
| Token transfers | `eth_subscribe("logs", {topics: [keccak256("Transfer(address,address,uint256)")]})` | Filter by `from` OR `to` matching tracked addresses |
| New blocks | `eth_subscribe("newHeads")` | None — all blocks |
| Pending transactions | `eth_subscribe("newPendingTransactions")` | Post-filter by `from` matching tracked addresses |
| Contract events | `eth_subscribe("logs", {address, topics})` | Protocol-specific contract addresses and event signatures |

**Optimization — Block-Driven Balance Updates**: Rather than subscribing to `logs` filtered to each tracked address (which creates N subscriptions for N addresses), subscribe to `newHeads` once per chain and on each new block, batch-fetch balances for all tracked addresses via `eth_getBalance` (JSON-RPC batch). This reduces subscription count to 1 per chain while ensuring per-block freshness. Token transfers still use `logs` subscriptions since they carry the transfer amount in the event data.

### Solana Integration: Subscription Service

sol-integration exposes a similar `SubscriptionService` using Solana-native subscription methods.

**SubscriptionService Interface**:

```
SubscriptionService {
  // Subscribe to SOL balance changes for tracked addresses
  subscribeAccountChanges(addresses: SolanaAddress[]): SubscriptionHandle

  // Subscribe to SPL token account changes for tracked addresses
  subscribeTokenAccounts(addresses: SolanaAddress[]): SubscriptionHandle

  // Subscribe to program account changes (for DeFi position tracking)
  subscribeProgramChanges(programId: ProgramId, filters: AccountFilter[]): SubscriptionHandle

  // Subscribe to slot changes (equivalent of new blocks)
  subscribeSlotChanges(): SubscriptionHandle

  // Subscribe to transaction signature status
  subscribeSignatureStatus(signature: TransactionSignature): SubscriptionHandle

  // Unsubscribe and clean up
  unsubscribe(handle: SubscriptionHandle): void

  // Get subscription health status
  getSubscriptionStatus(): SubscriptionStatus
}
```

**Underlying Solana WebSocket Methods Used**:

| Subscription Type | Solana WS Method | Parameters |
|-------------------|------------------|------------|
| SOL balance changes | `accountSubscribe(pubkey, {commitment: "confirmed"})` | One per tracked wallet address |
| SPL token changes | `accountSubscribe(tokenAccountPubkey, {commitment: "confirmed"})` | One per tracked token account (ATA) |
| DeFi positions | `programSubscribe(programId, {filters, commitment: "confirmed"})` | Protocol program ID + memcmp/dataSize filters |
| Slot updates | `slotSubscribe()` | None |
| Transaction status | `signatureSubscribe(signature, {commitment: "confirmed"})` | Auto-unsubscribes after finalized |

**Solana-Specific Consideration**: Unlike EVM where one `newHeads` subscription + batch `eth_getBalance` covers all addresses, Solana requires individual `accountSubscribe` calls per address. For wallets with many SPL token accounts (50+), this creates many subscriptions. However, Helius supports up to 40,000 concurrent subscriptions per connection, so this is not a practical limit for portfolio tracking use cases. The public Solana RPC WebSocket is heavily rate-limited and should NOT be used for subscriptions — always use Helius or Alchemy.

---

## 4. Portfolio Aggregation & CygnusWealthApp Consumption

### Portfolio Aggregation: Subscription Orchestrator

Portfolio-aggregation adds a `SubscriptionOrchestrator` that coordinates subscriptions across all integrations and pushes live updates to the experience layer.

**SubscriptionOrchestrator Responsibilities**:

1. **Lifecycle management**: When portfolio-aggregation receives tracked addresses from wallet-integration, it creates subscriptions via evm-integration and sol-integration SubscriptionServices
2. **Event normalization**: Subscription events from different chains arrive in chain-specific formats — the orchestrator normalizes them to unified `PortfolioUpdateEvent` domain events
3. **Deduplication**: Multiple subscription events can fire for the same logical update (e.g., a token transfer fires both a `Transfer` log event AND a balance change on `newHeads`). The orchestrator deduplicates within a 500ms window per address per chain
4. **Debouncing**: Rapid-fire subscription events (e.g., new blocks every 2s on Polygon, every 12s on Ethereum, every 0.4s on Solana) are debounced per-chain before triggering UI updates:
   - Solana: Debounce to 2-second windows (0.4s slots produce too many events)
   - Polygon: Debounce to 5-second windows
   - Ethereum/L2s: No debounce needed (12s+ block times)
5. **Aggregation trigger**: When a normalized event arrives, the orchestrator fetches updated data for the affected asset only (not full portfolio refresh), combines it with cached data for other assets, and emits an incremental `PortfolioUpdateEvent`

**Event Flow**:

```
evm-integration WS event (e.g., Transfer log on Ethereum)
  │
  ▼
SubscriptionOrchestrator.onIntegrationEvent()
  │
  ├── Normalize to PortfolioUpdateEvent
  ├── Deduplicate (500ms window)
  ├── Debounce (per-chain window)
  │
  ▼
SubscriptionOrchestrator.processUpdate()
  │
  ├── Fetch updated balance for affected address/token (HTTP, single call)
  ├── Merge with cached portfolio data
  ├── Update L1/L2 cache
  │
  ▼
Emit PortfolioUpdateEvent via domain event bus
  │
  ▼
CygnusWealthApp (Zustand store subscriber)
```

### CygnusWealthApp: Zustand Live Update Store

CygnusWealthApp adds a new **Live Update Store** alongside its existing UI State, User Configuration, and Cache stores.

**LiveUpdateStore**:

```
LiveUpdateStore {
  // State
  subscriptionStatus: Map<ChainId, 'active' | 'reconnecting' | 'failed' | 'polling'>
  lastUpdateTimestamp: Map<ChainId, number>
  pendingUpdates: PortfolioUpdateEvent[]  // Queued updates for batch rendering
  isLiveMode: boolean                      // true = WS active; false = polling fallback

  // Actions
  handlePortfolioUpdate(event: PortfolioUpdateEvent): void
  setSubscriptionStatus(chainId: ChainId, status: SubscriptionStatus): void
  enableLiveMode(): void
  disableLiveMode(): void
  flushPendingUpdates(): void
}
```

**Integration with Existing Stores**:

The LiveUpdateStore does NOT replace the Cache Store. Instead:

1. `LiveUpdateStore.handlePortfolioUpdate()` receives a `PortfolioUpdateEvent`
2. It updates the `CacheStore` with the new asset data (merging into existing portfolio)
3. It updates `lastUpdateTimestamp` for the affected chain
4. The `CacheStore` triggers React re-renders via Zustand's selector mechanism
5. UI components that select portfolio data from `CacheStore` automatically re-render with fresh values

**React Hook for Live Updates**:

```
useLivePortfolio() {
  // Subscribes to CacheStore for portfolio data
  // Subscribes to LiveUpdateStore for connection status
  // Returns: { portfolio, subscriptionStatus, isLive, lastUpdated }
}
```

**UI Indicators**:

- Green dot / "Live" badge when all chain subscriptions are active
- Yellow dot / "Reconnecting" when any chain is in reconnection
- Orange dot / "Polling" when fallen back to HTTP polling
- Red dot / "Offline" when no data source available
- Per-chain status visible in a connection status panel (settings page)

### Rendering Optimization

To prevent UI jitter from high-frequency updates:

1. **Batch rendering**: Collect all updates received within a 100ms window and apply them in a single React render cycle via `flushPendingUpdates()`
2. **Selective re-render**: Use Zustand selectors so only components displaying the affected asset re-render (not the entire portfolio view)
3. **Animation**: Balance changes animate (brief highlight or count-up animation) to give the user visual feedback that data is live

---

## 5. Polling Fallback

### When Polling Activates

Polling is the fallback, not the default. It activates under these conditions:

| Condition | Trigger | Polling Interval |
|-----------|---------|------------------|
| WebSocket connection fails after max retries (10) | `FAILED` state reached | 30 seconds |
| Provider does not support WebSocket (e.g., public RPCs) | No WS endpoint configured | 30 seconds |
| User is on a network that blocks WebSocket (corporate firewalls) | Connection timeout on WS handshake | 30 seconds |
| Browser tab is backgrounded (visibility API) | `document.hidden === true` | 60 seconds (reduced frequency to save quota) |
| Memory pressure detected | MemoryManager reports >70% usage | 60 seconds (close WS connections, switch to polling) |

### Polling Architecture

When polling is active, the existing HTTP-based data fetch pipeline is reused with a timer:

```
PollManager {
  // Per-chain poll timers
  timers: Map<ChainId, PollTimer>

  // Start polling for a chain (called when WS fails)
  startPolling(chainId: ChainId, interval: number): void

  // Stop polling for a chain (called when WS reconnects)
  stopPolling(chainId: ChainId): void

  // Poll cycle: fetch via HTTP, update cache, emit same PortfolioUpdateEvent
  pollCycle(chainId: ChainId): void
}
```

**Key design**: The polling fallback emits the same `PortfolioUpdateEvent` that WebSocket subscriptions emit. The CygnusWealthApp and portfolio-aggregation are transport-agnostic — they consume `PortfolioUpdateEvent` regardless of whether it originated from a WebSocket subscription or a poll cycle. The only difference visible to the UI is the `subscriptionStatus` indicator (green "Live" vs orange "Polling").

### Automatic Recovery

The PollManager continuously attempts to re-establish WebSocket connections while polling:

- Every 60 seconds while polling, attempt a WebSocket connection to the primary provider
- If successful, cancel poll timer, re-subscribe all subscriptions, emit `ConnectionRestored`
- If the secondary provider's WebSocket is available but primary is not, use secondary for WS
- WS recovery attempts use their own circuit breaker (separate from the HTTP circuit breaker) to avoid hammering a down WS endpoint

### Tab Visibility Optimization

When the browser tab is backgrounded:
1. Close WebSocket connections after 30 seconds of being hidden (saves server resources and client memory)
2. Switch to polling at 60-second intervals (just enough to keep cache warm)
3. On tab focus (`visibilitychange` event):
   - Immediately perform one HTTP poll to get latest data
   - Re-establish WebSocket connections
   - Resume live subscription mode

This prevents wasted resources on background tabs while ensuring the user sees fresh data immediately on returning.

---

## 6. Per-Chain WebSocket Provider Support

### Provider WebSocket Capabilities

Based on the RPC strategy, here is the WebSocket support matrix:

| Chain | Primary WS Provider | WS Endpoint Pattern | Secondary WS Provider | Public WS Fallback |
|-------|---------------------|---------------------|----------------------|---------------------|
| **Ethereum** | Alchemy | `wss://eth-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC `wss://lb.drpc.org/ogws?network=ethereum&dkey={key}` | None reliable |
| **Polygon** | Alchemy | `wss://polygon-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC `wss://lb.drpc.org/ogws?network=polygon&dkey={key}` | None reliable |
| **Arbitrum** | Alchemy | `wss://arb-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC | None reliable |
| **Optimism** | Alchemy | `wss://opt-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC | None reliable |
| **BNB Smart Chain** | Alchemy | `wss://bnb-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC | `wss://bsc-ws-node.nariox.org:443` (community, unreliable) |
| **Avalanche** | Alchemy | `wss://avax-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC | None reliable |
| **Base** | Alchemy | `wss://base-mainnet.g.alchemy.com/v2/{apiKey}` | dRPC | None reliable |
| **Fantom** | dRPC | `wss://lb.drpc.org/ogws?network=fantom&dkey={key}` | None (Alchemy deprecated Fantom) | `wss://wsapi.fantom.network` (community) |
| **Solana** | Helius | `wss://mainnet.helius-rpc.com/?api-key={apiKey}` | Alchemy `wss://solana-mainnet.g.alchemy.com/v2/{apiKey}` | `wss://api.mainnet-beta.solana.com` (rate-limited) |

### Provider-Specific Limitations

**Alchemy (EVM)**:
- Free tier: WebSocket connections included, same CU budget as HTTP
- `eth_subscribe` costs 10 CU per subscription creation, 0 CU per notification received
- Max 100 concurrent subscriptions per connection
- Connections idle >5 minutes are closed by server
- `newPendingTransactions` not available on all chains (check per-chain)

**dRPC (EVM)**:
- Free tier: WebSocket included
- Similar `eth_subscribe` support to standard Ethereum JSON-RPC
- Good for Fantom (only managed WS provider)
- Subscription limits vary by plan

**Helius (Solana)**:
- Free tier (10 RPS): WebSocket included, up to 40,000 subscriptions
- `accountSubscribe`, `programSubscribe`, `slotSubscribe`, `signatureSubscribe` all supported
- Gatekeeper Edge (Feb 2026) reduces connection latency 4-7x
- `logsSubscribe` supported for program log filtering
- Commitment levels: `processed`, `confirmed`, `finalized` — use `confirmed` for balance subscriptions (good balance of speed vs reliability)

**Alchemy (Solana)**:
- WebSocket support included
- `accountSubscribe`, `programSubscribe` supported
- Secondary Solana WS provider

### WebSocket Fallback Chain (Per-Chain)

When a WebSocket provider fails, the fallback is to the next WS-capable provider in the RPC strategy tier order, NOT directly to HTTP polling:

```
Primary WS Provider (Alchemy / Helius)
  ↓ WS connection failed
Secondary WS Provider (dRPC / Alchemy)
  ↓ WS connection failed
Tertiary WS Provider (if available)
  ↓ WS connection failed
HTTP Polling Fallback (using existing HTTP provider chain)
```

This means for most EVM chains, there are 2 WS providers to try before falling back to polling. For Solana, there are 3 (Helius → Alchemy → dRPC). For Fantom, there is only 1 managed WS provider (dRPC) before falling back to the community endpoint or polling.

---

## 7. Domain Event Additions

The following domain events are added to the Integration Domain event taxonomy (extending `IntegrationEventType` from `patterns.md`):

```
// WebSocket lifecycle events
WEBSOCKET_CONNECTED    = 'integration.websocket.connected'
WEBSOCKET_DISCONNECTED = 'integration.websocket.disconnected'
WEBSOCKET_RECONNECTING = 'integration.websocket.reconnecting'
WEBSOCKET_FAILED       = 'integration.websocket.failed'

// Subscription events
SUBSCRIPTION_CREATED   = 'integration.subscription.created'
SUBSCRIPTION_REMOVED   = 'integration.subscription.removed'
SUBSCRIPTION_ERROR     = 'integration.subscription.error'

// Transport fallback events
TRANSPORT_FALLBACK_TO_POLLING = 'integration.transport.fallback_polling'
TRANSPORT_RESTORED_TO_WS     = 'integration.transport.restored_ws'

// Live data events (emitted by both WS and polling paths)
LIVE_BALANCE_UPDATED   = 'integration.live.balance_updated'
LIVE_TRANSFER_DETECTED = 'integration.live.transfer_detected'
LIVE_BLOCK_RECEIVED    = 'integration.live.block_received'
```

These events flow through the existing domain event bus and are consumed by portfolio-aggregation's SubscriptionOrchestrator and CygnusWealthApp's LiveUpdateStore.

---

## 8. Impact on Existing Architecture

### Changes to contracts.md

The `BlockchainIntegrationContract` already includes `subscribeToUpdates(addresses): Subscription`. This directive defines what that subscription interface actually looks like. No contract interface change needed — only the implementation specification.

The `PortfolioAggregationContract` should add:

```
// Live data subscription
subscribeToLiveUpdates(): LiveSubscriptionHandle
getLiveConnectionStatus(): Map<ChainId, SubscriptionStatus>
```

### Changes to Integration Domain README

The "Future Expansion" section lists "WebSocket subscriptions for real-time updates" as an enhancement opportunity. This directive promotes it from future enhancement to core architecture. The README should be updated to list WebSocket as a primary transport, not a future item.

### Changes to rpc-strategy.md

Add a "WebSocket Endpoints" section alongside the existing HTTP endpoint tables, documenting the WS endpoint URLs per provider per chain (as specified in Section 6 above).

### Changes to evm-integration.md

The "Real-time Features" section already describes WebSocket subscriptions. This directive makes it prescriptive: add the `SubscriptionService` interface to the "Service Interface" section alongside BalanceService, TransactionService, and TrackingService.

### Changes to sol-integration.md

The "Event-Driven Updates" section describes notifications. This directive makes it prescriptive: add the `SubscriptionService` interface to the "Service Interface" section.

### Changes to cygnus-wealth-app.md

Add `LiveUpdateStore` to the State Management section. Add live status indicators to the UI Components section.

### Changes to portfolio-aggregation.md

Add `SubscriptionOrchestrator` to the Service Interface section alongside Portfolio Aggregation Service, Address Registry Service, and Sync Orchestrator.

---

## 9. Performance Impact

### Quota Savings

**Current state (polling every 30 seconds)**:
- 8 EVM chains × 1 balance query per tracked address per poll × 2 polls/minute = 16 RPC calls/minute per address per chain
- With 5 tracked addresses across 3 active chains: ~240 RPC calls/minute = ~14,400/hour = ~345,600/day
- At Alchemy's 1 CU per `eth_getBalance`: 345K CU/day = ~10.7M CU/month (35% of free tier burned on polling alone)

**With WebSocket architecture**:
- 1 `newHeads` subscription per chain = 3 subscriptions (30 CU one-time setup)
- Balance fetches only on new blocks: Ethereum ~300/hour, Polygon ~1800/hour, Arbitrum ~300/hour
- But: batch-fetch all 5 addresses per block = same call count BUT only when blocks actually arrive, not on a fixed timer
- Net reduction: ~40-60% fewer RPC calls, and zero calls during periods of no on-chain activity for tracked addresses
- Additional savings: Token transfer detection via `logs` subscription is 0 CU per notification (only subscription creation costs CU)

### Latency Improvement

| Metric | Polling (current) | WebSocket (proposed) |
|--------|-------------------|----------------------|
| Balance update latency | 0-30 seconds (avg 15s) | <2 seconds (next block) |
| Token transfer detection | 0-30 seconds (avg 15s) | <1 second (log event) |
| Pending tx visibility | Not detected until confirmed | Immediate (pending subscription) |
| New block awareness | 0-30 seconds (avg 15s) | <500ms |

---

## 10. Implementation Sequence

This is an architectural directive, not an implementation plan. However, the recommended implementation order is:

1. **WebSocket connection manager** in evm-integration (single chain: Ethereum)
2. **`newHeads` subscription** + block-driven balance refresh in evm-integration
3. **`logs` subscription** for ERC-20 Transfer events in evm-integration
4. **Polling fallback** with automatic WS recovery in evm-integration
5. **Extend to all EVM chains** (connection pool per chain)
6. **Solana `accountSubscribe`** + `programSubscribe` in sol-integration
7. **SubscriptionOrchestrator** in portfolio-aggregation (event normalization, deduplication, debouncing)
8. **LiveUpdateStore** in CygnusWealthApp (Zustand store + status indicators)
9. **Tab visibility optimization** (background/foreground transitions)
10. **DeFi position subscriptions** (contract event monitoring for vault/pool/staking contracts)

---

## Related Documentation

- **[RPC Node Strategy](../domains/integration/rpc-strategy.md)** — Provider endpoints and free tier budgets
- **[Integration Patterns](../domains/integration/patterns.md)** — Circuit breaker, retry, event publishing patterns
- **[Resilience & Performance](../domains/integration/resilience-performance.md)** — Connection pooling, keep-alive, fallback chain
- **[EVM Integration](../domains/integration/bounded-contexts/evm-integration.md)** — EVM-specific WebSocket features
- **[Solana Integration](../domains/integration/bounded-contexts/sol-integration.md)** — Solana-specific subscription methods
- **[Portfolio Aggregation](../domains/portfolio/bounded-contexts/portfolio-aggregation.md)** — Orchestration and event consumption
- **[CygnusWealth App](../domains/experience/bounded-contexts/cygnus-wealth-app.md)** — Zustand stores and UI state
- **[Domain Contracts](../contracts.md)** — Inter-domain communication contracts
