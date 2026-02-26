# en-fr0z: Multi-Wallet Multi-Account Architecture

**Status**: RECOMMENDATION
**Date**: 2026-02-18
**Bead**: en-fr0z
**Scope**: Cross-domain architecture for connecting multiple EVM wallet providers with multiple accounts per provider. Defines data model changes, contract amendments, and UX patterns across DataModels, WalletIntegrationSystem, EvmIntegration, PortfolioAggregation, and CygnusWealthApp.

---

## Executive Summary

**Users must be able to connect multiple wallet providers simultaneously and track all accounts exposed by each provider — not just the "active" one.**

The current WalletIntegrationSystem supports connecting multiple wallets, but the data model is flat: it tracks connections and addresses without a clear hierarchy linking addresses to their originating wallet provider. The portfolio system treats all addresses as an undifferentiated pool. Users with multiple wallets (MetaMask, Rabby, Crypto.com Onchain) and multiple accounts per wallet (e.g., 4 separate mnemonics imported into Crypto.com Onchain) cannot distinguish which holdings belong to which wallet or account, cannot label accounts, and cannot view portfolio breakdowns per wallet or per account.

This directive establishes the identity model, contract changes, and UX patterns for multi-wallet multi-account support across all affected bounded contexts.

### Key Constraint: Read-Only Account Discovery

We never access, derive, or handle mnemonics or private keys. Account discovery works through two mechanisms:

1. **Provider-exposed accounts**: The wallet provider (via EIP-1193 `eth_requestAccounts` or equivalent) tells us which addresses are available. We track all of them.
2. **User-added watch addresses**: Users can manually add any address for tracking — no wallet connection required, just read-only monitoring.

We cannot determine which mnemonic backs which address. We cannot derive HD paths. We see only what the wallet provider chooses to expose. This is by design — privacy-first, read-only.

---

## 1. Identity Model

### The Hierarchy

The multi-wallet multi-account model introduces a three-level identity hierarchy:

```
User
 └── WalletConnection[]  (one per connected wallet provider session)
       ├── provider: WalletProviderId      (metamask, rabby, crypto-com-onchain, etc.)
       ├── connectionId: string            (unique per connection session)
       ├── connectionLabel: string         (user-assigned: "My MetaMask", "Hardware Wallet")
       └── accounts: ConnectedAccount[]    (all addresses exposed by this provider)
             ├── address: string           (checksummed EVM address)
             ├── accountLabel: string      (user-assigned: "Main", "DeFi", "Cold Storage")
             ├── chainScope: ChainId[]     (chains this address is tracked on)
             └── source: 'provider' | 'manual'  (how this address was added)

 └── WatchAddress[]  (addresses not connected via any wallet — read-only monitoring)
       ├── address: string
       ├── addressLabel: string
       └── chainScope: ChainId[]
```

### Why This Hierarchy

**WalletConnection is not just a provider type — it's a session.** A user might connect MetaMask twice (e.g., different browser profiles via WalletConnect) or have two separate Ledger devices via two WalletConnect sessions. Each connection is a distinct session with its own set of exposed accounts.

**ConnectedAccount is an address within a connection.** When Crypto.com Onchain exposes 4 addresses from 4 imported mnemonics, those are 4 ConnectedAccounts under one WalletConnection. We don't know (or care) that they come from different mnemonics — we only know the wallet provider told us about them.

**WatchAddress is independent of any wallet connection.** Users can paste any address to track without connecting a wallet. These addresses have no associated provider, no session lifecycle, and never receive `accountsChanged` events. They are purely passive tracking targets.

### Provider-Specific Account Exposure Behavior

Understanding how each wallet provider exposes accounts is critical for the connection flow:

| Wallet Provider | Account Exposure via EIP-1193 | Multi-Account Behavior |
|----------------|-------------------------------|------------------------|
| **MetaMask** | `eth_requestAccounts` returns only the currently selected account. Additional accounts surface via `accountsChanged` event when user switches in the wallet UI. | User must manually switch accounts in MetaMask. We detect each switch and accumulate discovered accounts. |
| **Rabby** | `eth_requestAccounts` can return multiple accounts if user grants permission. | Rabby natively supports multi-account exposure. We receive all granted accounts at once. |
| **Crypto.com Onchain** | `eth_requestAccounts` returns the active account. `accountsChanged` fires on switch. Multiple imported mnemonics appear as separate accounts in the wallet UI. | Similar to MetaMask — user switches in wallet, we accumulate. The wallet manages mnemonic-to-account mapping internally. |
| **WalletConnect** | Session approval returns all accounts the user selected. | User selects which accounts to share during the pairing flow. We receive the full selection. |
| **Coinbase Wallet** | `eth_requestAccounts` returns the active account. | Similar to MetaMask — single active account model. |
| **Trust Wallet** | `eth_requestAccounts` returns the active account. | Single active account model. |
| **Frame** | `eth_requestAccounts` returns the active account. Supports hot, hardware, and smart accounts. | Account switching via Frame's own UI. |

### Account Accumulation Strategy

Because most EIP-1193 wallets only expose the currently selected account, we use an **accumulation model**:

1. On `connectWallet()`, we receive the initial account(s) from `eth_requestAccounts`.
2. We store these accounts in the connection's account list.
3. When `accountsChanged` fires (user switched accounts in their wallet), we **add** the new account to the list — we do NOT replace the existing accounts.
4. An account is only removed from the list if the user explicitly removes it in our UI, or the entire wallet connection is disconnected.
5. The "active" account (the one currently selected in the wallet provider) is tracked as a separate property — it affects which account is highlighted in the UI, but all accumulated accounts remain tracked for portfolio purposes.

This means: if a user connects MetaMask, switches between 3 accounts over the course of a session, we track all 3 addresses — even though MetaMask only ever "shows" one at a time.

### Account Persistence

Account lists are persisted alongside connection sessions in encrypted local storage. When the user returns and the wallet session is restored:

1. Restore the WalletConnection from persisted data.
2. Re-request accounts from the provider via `eth_requestAccounts`.
3. Merge: any new accounts from the provider are added; any previously tracked accounts that the provider no longer exposes are marked `stale` (not removed — the user may have simply not selected them yet).
4. Stale accounts remain tracked for portfolio purposes but are flagged in the UI as "not currently active in wallet."

---

## 2. Data Model Changes (Contract Domain — data-models)

### New Types

**`WalletProviderId`** — canonical identifier for a wallet provider:
- Values: `'metamask'`, `'rabby'`, `'walletconnect'`, `'coinbase-wallet'`, `'trust-wallet'`, `'frame'`, `'crypto-com-onchain'`, `'phantom'`, `'solflare'`, `'backpack'`, `'exodus'`, `'manual'`
- String union type — extensible without breaking changes when new wallets are supported

**`WalletConnectionId`** — unique identifier for a specific wallet connection session:
- Format: `{providerId}:{randomId}` (e.g., `metamask:a1b2c3d4`)
- The randomId component ensures uniqueness when the same provider is connected multiple times (e.g., two WalletConnect sessions)

**`AccountId`** — unique identifier for a specific account across the system:
- Format: `{walletConnectionId}:{checksummedAddress}` (e.g., `metamask:a1b2c3d4:0xAbC...123`)
- For watch addresses: `watch:{checksummedAddress}`
- This is the **primary key** used throughout the system to reference a specific account. It disambiguates the same address appearing in different wallet connections (e.g., imported into both MetaMask and Rabby)

**`WalletConnection`** — a connected wallet provider session:
- `connectionId: WalletConnectionId`
- `providerId: WalletProviderId`
- `providerName: string` — human-readable provider name (e.g., "MetaMask")
- `providerIcon: string` — URL or identifier for the provider's icon
- `connectionLabel: string` — user-assigned label (default: provider name)
- `accounts: ConnectedAccount[]`
- `activeAccountAddress: string | null` — currently selected account in the wallet provider
- `supportedChains: ChainId[]` — chains this wallet connection supports
- `connectedAt: string` — ISO 8601 timestamp of initial connection
- `lastActiveAt: string` — ISO 8601 timestamp of last activity
- `sessionStatus: 'active' | 'stale' | 'disconnected'`

**`ConnectedAccount`** — a single account within a wallet connection:
- `accountId: AccountId`
- `address: string` — checksummed address
- `accountLabel: string` — user-assigned label (default: truncated address)
- `chainScope: ChainId[]` — chains to track this account on (default: all supported by the connection)
- `source: 'provider' | 'manual'` — how this account was added (`provider` = discovered via EIP-1193; `manual` = user typed the address into this connection's account list)
- `discoveredAt: string` — ISO 8601 timestamp when first seen
- `isStale: boolean` — true if the provider no longer includes this account in `eth_requestAccounts` responses
- `isActive: boolean` — true if this is the currently selected account in the wallet provider

**`WatchAddress`** — an address tracked independently of any wallet connection:
- `accountId: AccountId` — format: `watch:{checksummedAddress}`
- `address: string` — checksummed address
- `addressLabel: string` — user-assigned label
- `chainScope: ChainId[]` — chains to track
- `addedAt: string` — ISO 8601 timestamp

**`AccountGroup`** — user-defined grouping of accounts for organizational purposes:
- `groupId: string`
- `groupName: string` — user-assigned (e.g., "DeFi Accounts", "Long-term Holdings", "Family")
- `accountIds: AccountId[]` — references to accounts from any connection or watch addresses
- `createdAt: string` — ISO 8601 timestamp

### Updated Types

**`Portfolio`** — add account attribution:
- New field: `accountBreakdown: Map<AccountId, AccountPortfolio>` — per-account portfolio breakdown
- New field: `walletBreakdown: Map<WalletConnectionId, WalletPortfolio>` — per-wallet portfolio breakdown
- Existing `assets`, `totalValue`, etc. remain as aggregate across all accounts

**`AccountPortfolio`** — portfolio slice for a single account:
- `accountId: AccountId`
- `accountLabel: string`
- `walletConnectionId: WalletConnectionId | 'watch'`
- `providerName: string`
- `assets: Asset[]`
- `totalValue: Value`
- `lastUpdated: string`

**`WalletPortfolio`** — portfolio slice for all accounts in a wallet connection:
- `walletConnectionId: WalletConnectionId`
- `connectionLabel: string`
- `providerName: string`
- `accounts: AccountPortfolio[]`
- `totalValue: Value`

**`Asset`** — add account attribution:
- New field: `accountId: AccountId` — which account holds this asset
- New field: `walletConnectionId: WalletConnectionId | 'watch'` — shortcut to the wallet connection
- Existing fields unchanged

**`Transaction`** — add account context:
- New field: `accountId: AccountId` — which account this transaction belongs to
- New field: `walletConnectionId: WalletConnectionId | 'watch'`
- Existing fields unchanged

**`IntegrationSource`** — extended to include account context:
- New field: `accountId?: AccountId` — when the data is account-specific
- Existing fields unchanged

### What Does NOT Change

- `Balance`, `Value`, `MarketData`, `PriceHistory` — these are account-agnostic by nature
- `ChainRpcConfig`, `RpcProviderConfig` — RPC configuration is independent of account identity
- `DeFiPosition` — already includes `ownerAddress`; add `accountId` alongside for direct lookup

---

## 3. WalletIntegrationSystem Contract Changes

### Current State

The WalletIntegrationSystem exposes two contracts:

1. `WalletConnectionContract` — for CygnusWealthApp to manage connections
2. `WalletIntegrationContract` — for PortfolioAggregation to get addresses

Both treat connections and addresses as flat lists without hierarchy.

### Amended WalletConnectionContract (App → WalletIntegration)

```
WalletConnectionContract {
  // Commands — Connection Lifecycle
  connectWallet(type: WalletProviderId, options?: ConnectionOptions): WalletConnection
  disconnectWallet(connectionId: WalletConnectionId): void
  switchChain(connectionId: WalletConnectionId, chainId: ChainId): void

  // Commands — Account Management
  setConnectionLabel(connectionId: WalletConnectionId, label: string): void
  setAccountLabel(accountId: AccountId, label: string): void
  setAccountChainScope(accountId: AccountId, chains: ChainId[]): void
  removeAccount(accountId: AccountId): void
  addManualAccountToConnection(connectionId: WalletConnectionId, address: string): ConnectedAccount

  // Commands — Watch Addresses
  addWatchAddress(address: string, label: string, chains: ChainId[]): WatchAddress
  removeWatchAddress(accountId: AccountId): void
  updateWatchAddress(accountId: AccountId, label?: string, chains?: ChainId[]): void

  // Commands — Account Groups
  createAccountGroup(name: string, accountIds: AccountId[]): AccountGroup
  updateAccountGroup(groupId: string, name?: string, accountIds?: AccountId[]): void
  deleteAccountGroup(groupId: string): void

  // Queries
  getAvailableWallets(): WalletProviderInfo[]
  getConnections(): WalletConnection[]
  getConnection(connectionId: WalletConnectionId): WalletConnection
  getAccountsByConnection(connectionId: WalletConnectionId): ConnectedAccount[]
  getWatchAddresses(): WatchAddress[]
  getAccountGroups(): AccountGroup[]
  getAllTrackedAccounts(): (ConnectedAccount | WatchAddress)[]

  // Events
  onWalletConnected: Event<WalletConnection>
  onWalletDisconnected: Event<WalletConnectionId>
  onAccountDiscovered: Event<{connectionId: WalletConnectionId, account: ConnectedAccount}>
  onAccountRemoved: Event<AccountId>
  onActiveAccountChanged: Event<{connectionId: WalletConnectionId, address: string}>
  onChainChanged: Event<{connectionId: WalletConnectionId, chainId: ChainId}>
  onAccountLabelChanged: Event<{accountId: AccountId, label: string}>
  onSessionRestored: Event<WalletConnection>
  onSessionStale: Event<WalletConnectionId>
}
```

### Amended WalletIntegrationContract (PortfolioAggregation → WalletIntegration)

```
WalletIntegrationContract {
  // Queries — Account-Aware
  getTrackedAddresses(): TrackedAddress[]
  getTrackedAddressesByChain(chain: ChainId): TrackedAddress[]
  getTrackedAddressesByWallet(connectionId: WalletConnectionId): TrackedAddress[]
  getTrackedAddressesByGroup(groupId: string): TrackedAddress[]
  getAccountMetadata(accountId: AccountId): AccountMetadata

  // Events
  onAddressAdded: Event<TrackedAddress>
  onAddressRemoved: Event<AccountId>
  onAddressChainScopeChanged: Event<{accountId: AccountId, chains: ChainId[]}>
}
```

Where `TrackedAddress` is:

```
TrackedAddress {
  accountId: AccountId
  address: string
  walletConnectionId: WalletConnectionId | 'watch'
  providerId: WalletProviderId | 'watch'
  accountLabel: string
  connectionLabel: string
  chainScope: ChainId[]
}
```

And `AccountMetadata` is:

```
AccountMetadata {
  accountId: AccountId
  address: string
  accountLabel: string
  connectionLabel: string
  providerId: WalletProviderId | 'watch'
  walletConnectionId: WalletConnectionId | 'watch'
  groups: string[]     // group IDs this account belongs to
  discoveredAt: string
  isStale: boolean
  isActive: boolean
}
```

### Connection Flow Changes

#### Initial Connection

1. User clicks "Connect Wallet" and selects a provider (e.g., MetaMask)
2. WalletIntegrationSystem calls the provider's EIP-1193 `eth_requestAccounts`
3. Provider returns account(s) — typically one for MetaMask, potentially multiple for Rabby
4. System creates a `WalletConnection` with a new `connectionId` and `ConnectedAccount` entries for each returned address
5. System emits `onWalletConnected` with the full `WalletConnection`
6. System emits `onAccountDiscovered` for each account

#### Account Accumulation

1. User switches accounts in their wallet provider (e.g., switches from Account 1 to Account 2 in MetaMask)
2. Provider fires `accountsChanged` event with the new active account
3. WalletIntegrationSystem checks if this address already exists in the connection's account list
4. If new: creates a `ConnectedAccount`, adds to the connection, emits `onAccountDiscovered`
5. If existing: updates `isActive` flags on all accounts in the connection
6. Emits `onActiveAccountChanged`

#### Watch Address Addition

1. User navigates to "Add Watch Address" in the UI
2. User enters an address and optional label
3. System validates the address format (checksumming for EVM)
4. System creates a `WatchAddress` entry
5. Emits `onAddressAdded` with the `TrackedAddress`

#### Session Restoration

1. App starts, reads persisted connection data from encrypted local storage
2. For each persisted `WalletConnection`:
   a. Check if the wallet provider is still available (injected into `window.ethereum` or provider registry)
   b. If available: attempt to re-request accounts via `eth_requestAccounts` (some wallets support silent reconnection)
   c. Merge provider-returned accounts with persisted account list (accumulate, never subtract)
   d. Mark accounts not returned by provider as `isStale: true`
   e. Emit `onSessionRestored`
3. If the provider is not available (extension uninstalled, different browser):
   a. Mark the connection as `sessionStatus: 'stale'`
   b. Keep all accounts tracked (they still have valid addresses for read-only monitoring)
   c. Emit `onSessionStale`
   d. The portfolio continues to track these addresses — they function like watch addresses until the wallet is reconnected

### Multi-Provider Conflict Resolution

When the same address appears in multiple wallet connections (e.g., user imported the same seed into both MetaMask and Rabby):

- **Both connections track the address independently.** Each connection has its own `ConnectedAccount` with a distinct `AccountId` (since `AccountId` includes the `connectionId`).
- **Portfolio data is NOT duplicated.** The PortfolioAggregation layer deduplicates by address when computing aggregate portfolio values (see Section 5).
- **The UI shows the account under both wallets** in wallet management views, but portfolio values are attributed once per unique address.
- **The user can remove the duplicate** from one connection if they prefer, or keep both for organizational clarity.

### Security Considerations

- **No private key access**: `eth_requestAccounts` and `accountsChanged` only return public addresses. No method in the EIP-1193 subset we use accesses private keys.
- **No transaction preparation**: We never call `eth_sendTransaction`, `personal_sign`, or any signing methods.
- **Minimal permissions**: We request only `eth_accounts` permission. No additional permissions (e.g., `eth_sign`) are requested.
- **Session data encryption**: All persisted connection and account data is encrypted with the user's password via the existing EncryptionService (AES-256).

---

## 4. EvmIntegration Contract Changes

### Current State

The `BlockchainIntegrationContract` already accepts arrays of addresses:

```
getBalances(addresses): BalanceList
getTransactions(addresses, options): TransactionList
getTokens(addresses): TokenList
getNFTs(addresses): NFTList
getDeFiPositions(addresses): DeFiPositionList
subscribeToUpdates(addresses): Subscription
```

This interface is already multi-address capable. The changes needed are about **attribution** and **efficiency**, not fundamental interface redesign.

### Amended BlockchainIntegrationContract

```
BlockchainIntegrationContract {
  // Queries — now return account-attributed results
  getBalances(requests: AddressRequest[]): AccountBalanceList
  getTransactions(requests: AddressRequest[], options: TxOptions): AccountTransactionList
  getTokens(requests: AddressRequest[]): AccountTokenList
  getNFTs(requests: AddressRequest[]): AccountNFTList
  getDeFiPositions(requests: AddressRequest[]): AccountDeFiPositionList

  // Real-time — subscribe per-account
  subscribeToUpdates(requests: AddressRequest[]): AccountSubscription

  // Batch optimization
  getBatchBalances(requests: AddressRequest[]): AccountBalanceList
}
```

Where `AddressRequest` is:

```
AddressRequest {
  accountId: AccountId
  address: string
  chainScope: ChainId[]   // which chains to query for this address
}
```

And `AccountBalanceList` is:

```
AccountBalanceList {
  balances: AccountBalance[]
  errors: AccountError[]      // per-account errors (partial failure)
  timestamp: string
}

AccountBalance {
  accountId: AccountId
  address: string
  chainId: ChainId
  nativeBalance: Balance
  tokenBalances: TokenBalance[]
}
```

The same pattern applies to `AccountTransactionList`, `AccountTokenList`, etc. — each result item carries its `accountId` for attribution back to the originating account.

### Why AddressRequest Instead of Plain Addresses

The switch from `string[]` addresses to `AddressRequest[]` serves three purposes:

1. **Attribution**: Results carry `accountId` back, so the portfolio layer can attribute holdings to specific accounts without reverse-lookups.
2. **Per-account chain scoping**: Different accounts may track different chains. An Ethereum-only cold storage address doesn't need Polygon queries.
3. **Deduplication at the integration level**: If the same address appears in two wallet connections (two different `accountId`s), evm-integration can detect the duplicate address, query it once, and return the result attributed to both `accountId`s. This prevents redundant RPC calls.

### Address Deduplication in Batch Queries

When evm-integration receives `AddressRequest[]` with duplicate addresses (same address, different `accountId`s):

1. Group requests by unique `(address, chainId)` pairs.
2. Execute one RPC query per unique `(address, chainId)`.
3. Fan out the result: for each `accountId` that maps to that address, create an `AccountBalance` entry with the same data but the correct `accountId`.

This ensures we never query the same address twice on the same chain, regardless of how many wallet connections reference it.

### Subscription Changes

The existing `SubscriptionService` (from en-p0ha) already subscribes by address. The change is that subscription events now carry `accountId` context:

```
SubscriptionEvent {
  // existing fields
  chainId: ChainId
  address: string
  type: SubscriptionType
  data: any

  // new field
  accountIds: AccountId[]   // all account IDs associated with this address
}
```

`accountIds` is plural because the same address may belong to multiple wallet connections. The subscription fires once per on-chain event, and the portfolio layer resolves which accounts to attribute it to.

### Performance Impact

With N wallet connections and M accounts each, the worst case is N×M addresses to track. However:

- **Address deduplication** means unique addresses is typically much less than total AccountIds.
- **Chain scoping** means each address is only queried on its relevant chains.
- **Batch queries** (already standard from existing architecture) fetch all addresses per chain in a single batch call.
- **WebSocket subscriptions** (from en-p0ha) share one `newHeads` subscription per chain regardless of account count — balance batch-fetch on new block scales linearly with unique addresses but is a single round of RPC calls.

**Practical scale**: A power user with 5 wallet providers, 4 accounts each = 20 account IDs. With ~50% address overlap (same seed in multiple wallets), that's ~10-12 unique addresses. At 8 EVM chains, that's ~80-96 `(address, chain)` balance queries per refresh — well within the existing batch optimization and rate limit budgets.

---

## 5. PortfolioAggregation Contract Changes

### Current State

PortfolioAggregation has:
- `Portfolio Aggregation Service` — fetches from all sources, combines, deduplicates
- `Address Registry Service` — manages tracked addresses
- `Sync Orchestrator` — coordinates refresh cycles
- `SubscriptionOrchestrator` (from en-p0ha) — coordinates live updates

The current model treats all addresses as a flat pool and returns a single aggregate portfolio.

### Amended PortfolioAggregationContract (App → PortfolioAggregation)

```
PortfolioAggregationContract {
  // Commands
  refreshPortfolio(): Portfolio
  refreshAccount(accountId: AccountId): AccountPortfolio
  refreshWallet(connectionId: WalletConnectionId): WalletPortfolio
  addIntegration(type, config): void
  removeIntegration(id): void

  // Queries — Aggregate
  getPortfolio(): Portfolio
  getAssets(): AssetList
  getTransactions(filter: TransactionFilter): TransactionList

  // Queries — Account-Scoped
  getAccountPortfolio(accountId: AccountId): AccountPortfolio
  getWalletPortfolio(connectionId: WalletConnectionId): WalletPortfolio
  getGroupPortfolio(groupId: string): GroupPortfolio
  getAssetsByAccount(accountId: AccountId): AssetList
  getTransactionsByAccount(accountId: AccountId, filter?: TransactionFilter): TransactionList

  // Queries — Cross-Account Analysis
  getAssetDistribution(symbol: string): AssetDistribution
  getAccountSummaries(): AccountSummary[]

  // Live Updates
  subscribeToLiveUpdates(): LiveSubscriptionHandle
  getLiveConnectionStatus(): Map<ChainId, SubscriptionStatus>

  // Events
  onPortfolioUpdated: Event<Portfolio>
  onAccountPortfolioUpdated: Event<{accountId: AccountId, portfolio: AccountPortfolio}>
  onSyncComplete: Event
  onError: Event
}
```

Where `TransactionFilter` is extended:

```
TransactionFilter {
  // existing filter fields
  chains?: ChainId[]
  types?: TransactionType[]
  dateRange?: DateRange

  // new account-scoped filters
  accountIds?: AccountId[]
  walletConnectionIds?: WalletConnectionId[]
  groupIds?: string[]
}
```

And `AssetDistribution` is:

```
AssetDistribution {
  symbol: string
  totalQuantity: Balance
  totalValue: Value
  distribution: AccountAssetEntry[]
}

AccountAssetEntry {
  accountId: AccountId
  accountLabel: string
  connectionLabel: string
  quantity: Balance
  value: Value
  percentage: number    // percentage of total for this asset
}
```

And `AccountSummary` is:

```
AccountSummary {
  accountId: AccountId
  accountLabel: string
  connectionLabel: string
  providerId: WalletProviderId | 'watch'
  totalValue: Value
  assetCount: number
  chains: ChainId[]
  lastUpdated: string
}
```

And `GroupPortfolio` is:

```
GroupPortfolio {
  groupId: string
  groupName: string
  accounts: AccountPortfolio[]
  totalValue: Value
  lastUpdated: string
}
```

### Address Registry → Account Registry

The `Address Registry Service` is renamed to `Account Registry Service` to reflect its expanded role:

```
AccountRegistryService {
  // Replaces AddressRegistryService
  registerAccount(trackedAddress: TrackedAddress): void
  unregisterAccount(accountId: AccountId): void
  updateAccountChainScope(accountId: AccountId, chains: ChainId[]): void

  // Queries
  getAllTrackedAddresses(): TrackedAddress[]
  getTrackedAddressesByChain(chain: ChainId): TrackedAddress[]
  getUniqueAddressesByChain(chain: ChainId): string[]   // deduplicated for RPC efficiency
  getAccountIdsByAddress(address: string): AccountId[]   // reverse lookup: address → all account IDs

  // Account-Address mapping (for deduplication)
  resolveAccountIds(address: string, chainId: ChainId): AccountId[]
}
```

The `resolveAccountIds` method is critical for the deduplication flow: when evm-integration returns data for an address, the Account Registry resolves which `AccountId`s that address maps to, enabling proper attribution.

### Deduplication Rules — Updated

The existing deduplication rules (from portfolio-aggregation.md) handle the same asset from multiple data sources (e.g., ETH balance from both on-chain and Coinbase). Multi-account adds a new dimension:

**Same address, multiple AccountIds** (address imported into two wallets):
- This is NOT a portfolio duplicate. It represents the same on-chain holdings.
- The aggregate portfolio counts this asset ONCE (deduplicated by address).
- The per-account views show the asset under EACH account that references this address (for organizational clarity).
- The `Portfolio.accountBreakdown` attributes the value to each `AccountId`, but `Portfolio.totalValue` does not double-count.

**Different addresses, same asset** (e.g., USDC on two different addresses):
- This IS two separate holdings. Not deduplicated.
- Both appear in the aggregate portfolio as separate line items (or summed by asset symbol with per-account breakdown).

**Cross-source: on-chain address also tracked in CEX** (e.g., user's deposit address on Coinbase is also tracked as a watch address):
- Existing rule applies: on-chain data preferred over CEX data.
- Deduplication by address: if the CEX reports a balance for the same address that on-chain data covers, on-chain data wins.
- CEX-specific assets (fiat balances, earn positions) that don't have on-chain equivalents are always included.

### Aggregation Modes

PortfolioAggregation supports three aggregation modes, selectable by the caller:

1. **Aggregate** (default): All unique addresses across all accounts. Deduplicated by address. Single combined portfolio. This is `getPortfolio()`.

2. **Per-Account**: Assets attributed to each `AccountId`. Same address appearing in multiple accounts shows in each. This is `getAccountPortfolio(accountId)`.

3. **Per-Wallet**: Assets grouped by `WalletConnectionId`. Rollup of all accounts in a wallet connection. This is `getWalletPortfolio(connectionId)`.

4. **Per-Group**: Assets grouped by user-defined `AccountGroup`. Rollup of all accounts in the group. This is `getGroupPortfolio(groupId)`.

### SubscriptionOrchestrator Changes

The SubscriptionOrchestrator (from en-p0ha) manages live updates. Multi-account adds account attribution to the event flow:

**Updated Event Flow**:

```
evm-integration WS event
  │
  ▼
SubscriptionOrchestrator.onIntegrationEvent()
  │
  ├── Normalize to PortfolioUpdateEvent (existing)
  ├── Deduplicate (500ms window) (existing)
  ├── Debounce (per-chain window) (existing)
  ├── Resolve accountIds from address via AccountRegistry   ← NEW
  │
  ▼
SubscriptionOrchestrator.processUpdate()
  │
  ├── Fetch updated balance for affected address (existing)
  ├── Attribute result to all resolved accountIds           ← NEW
  ├── Update aggregate portfolio cache (existing)
  ├── Update per-account portfolio caches                   ← NEW
  │
  ▼
Emit PortfolioUpdateEvent (now includes accountIds)
  │
  ▼
CygnusWealthApp
```

**Updated PortfolioUpdateEvent**:

```
PortfolioUpdateEvent {
  // existing fields
  chainId: ChainId
  address: string
  updateType: 'balance' | 'token' | 'nft' | 'defi' | 'transaction'
  data: any
  timestamp: string

  // new fields
  accountIds: AccountId[]          // all accounts associated with this address
  walletConnectionIds: (WalletConnectionId | 'watch')[]  // all wallet connections affected
}
```

---

## 6. CygnusWealthApp UX Patterns

### State Management Changes

#### Updated User Configuration Store

The existing User Configuration Store manages tracked addresses and API keys. Multi-account restructures this:

```
UserConfigurationStore {
  // Wallet Connections (replaces flat address list)
  walletConnections: Map<WalletConnectionId, WalletConnection>
  watchAddresses: Map<AccountId, WatchAddress>
  accountGroups: Map<string, AccountGroup>

  // Account label management
  accountLabels: Map<AccountId, string>      // overrides default labels
  connectionLabels: Map<WalletConnectionId, string>

  // CEX integrations (unchanged)
  cexApiKeys: Map<string, EncryptedApiKey>

  // Actions
  addWalletConnection(connection: WalletConnection): void
  removeWalletConnection(connectionId: WalletConnectionId): void
  updateAccountLabel(accountId: AccountId, label: string): void
  updateConnectionLabel(connectionId: WalletConnectionId, label: string): void
  addWatchAddress(address: WatchAddress): void
  removeWatchAddress(accountId: AccountId): void
  createGroup(name: string, accountIds: AccountId[]): void
  updateGroup(groupId: string, updates: Partial<AccountGroup>): void
  deleteGroup(groupId: string): void
  setAccountChainScope(accountId: AccountId, chains: ChainId[]): void
}
```

#### Updated Cache Store

```
CacheStore {
  // existing
  aggregatePortfolio: Portfolio | null
  lastRefreshTimestamp: number

  // new — per-account caching
  accountPortfolios: Map<AccountId, AccountPortfolio>
  walletPortfolios: Map<WalletConnectionId, WalletPortfolio>

  // Actions
  updateAggregatePortfolio(portfolio: Portfolio): void
  updateAccountPortfolio(accountId: AccountId, portfolio: AccountPortfolio): void
  invalidateAccount(accountId: AccountId): void
  invalidateWallet(connectionId: WalletConnectionId): void
}
```

#### Updated LiveUpdateStore

The LiveUpdateStore (from en-p0ha) adds account-awareness:

```
LiveUpdateStore {
  // existing
  subscriptionStatus: Map<ChainId, SubscriptionStatus>
  lastUpdateTimestamp: Map<ChainId, number>
  pendingUpdates: PortfolioUpdateEvent[]
  isLiveMode: boolean

  // new
  lastAccountUpdateTimestamp: Map<AccountId, number>

  // updated action
  handlePortfolioUpdate(event: PortfolioUpdateEvent): void
    // now updates both CacheStore.aggregatePortfolio and CacheStore.accountPortfolios
    // for each accountId in the event
}
```

### New React Hooks

```
useWalletConnections()
  // Returns: { connections, connect, disconnect, isConnecting }

useAccounts(filter?: { connectionId?, groupId? })
  // Returns: { accounts, watchAddresses, allTracked }

useAccountPortfolio(accountId: AccountId)
  // Returns: { portfolio, isLoading, lastUpdated, refresh }

useWalletPortfolio(connectionId: WalletConnectionId)
  // Returns: { portfolio, isLoading, lastUpdated, refresh }

useGroupPortfolio(groupId: string)
  // Returns: { portfolio, isLoading, lastUpdated, refresh }

useAssetDistribution(symbol: string)
  // Returns: { distribution, totalQuantity, totalValue }

useAccountSummaries()
  // Returns: { summaries, totalValue, accountCount }

usePortfolioView(mode: 'aggregate' | 'by-account' | 'by-wallet' | 'by-group')
  // Returns the appropriate portfolio view based on mode
```

### UI Component Patterns

#### Wallet Management Panel

Located in Settings/Configuration, this is where users manage their wallet connections:

**Wallet Connection List**:
- Lists all connected wallets with provider icon, connection label, and account count
- Each wallet expands to show its accounts with addresses and labels
- "Connect Wallet" button triggers provider selection modal
- Per-connection actions: rename, disconnect, add manual address

**Account Management** (within each wallet connection):
- List of accumulated accounts with labels, addresses, and chain scope
- Inline label editing (click to edit)
- Chain scope selector (checkboxes for which chains to track)
- "Active" indicator for the currently selected account in the wallet
- "Stale" indicator for accounts not currently exposed by the provider
- Remove button (removes from tracking, not from the wallet itself)

**Watch Address Section**:
- List of watch-only addresses with labels
- "Add Watch Address" button with address input and label field
- Per-address actions: rename, update chain scope, remove

**Account Groups Section**:
- List of user-created groups with member counts and total values
- "Create Group" button with name input and account selector
- Drag-and-drop account assignment
- Per-group actions: rename, edit members, delete

#### Portfolio Dashboard Enhancements

**View Mode Selector**: A segmented control at the top of the portfolio dashboard:
- **All** (default): Aggregate portfolio across all accounts
- **By Wallet**: Grouped by wallet connection
- **By Account**: Individual account breakdowns
- **By Group**: Grouped by user-defined groups (only shown if groups exist)

**Account Filter Chip Bar**: Below the view mode selector, a horizontal scrollable bar of filter chips:
- One chip per account/wallet/group (depending on current view mode)
- Click to toggle inclusion/exclusion
- "Select All" / "Clear" actions
- Active filters persist across navigation within the session

**Asset Rows — Account Attribution**:
- In aggregate view: asset rows show total holdings. Expanding a row reveals per-account breakdown.
- In per-account view: asset rows show only that account's holdings.
- Each asset row includes a small wallet icon indicating the source provider.

**Portfolio Summary Cards**:
- In aggregate view: single card with total value
- In by-wallet view: one card per wallet with total value and account count
- In by-account view: one card per account with individual value
- Cards include provider icon, label, and value

#### Wallet Connection Modal

Triggered by "Connect Wallet":

1. **Provider Selection**: Grid of available wallet provider icons (detected from `window.ethereum` and registered providers)
2. **Connection**: Provider-specific connection flow (MetaMask popup, WalletConnect QR code, etc.)
3. **Account Review**: After connection, show discovered accounts with default labels. User can edit labels before confirming.
4. **Chain Scope**: Select which chains to track for the new connection (default: all supported)
5. **Confirmation**: Summary of connection with account list and chain scope

#### Account Discovery Prompt

When `accountsChanged` fires and a new account is detected:

- A non-intrusive toast notification appears: "New account detected in {walletName}: {truncatedAddress}"
- Clicking the toast opens a small panel to name the account and confirm chain scope
- If dismissed, the account is still tracked with the default label (truncated address)
- The notification does NOT block UI interaction

---

## 7. Cross-Domain Contract Impact Summary

### contracts.md Amendments

| Contract | Change | Details |
|----------|--------|---------|
| App → Wallet Integration | **Major revision** | `WalletConnectionContract` expanded with account management, watch address, and group commands |
| Portfolio Aggregation → Wallet Integration | **Major revision** | `WalletIntegrationContract` now returns `TrackedAddress[]` with full account context |
| Portfolio Aggregation → EVM/Sol Integration | **Moderate revision** | `BlockchainIntegrationContract` methods accept `AddressRequest[]` instead of `string[]`; results carry `accountId` |
| App → Portfolio Aggregation | **Major revision** | `PortfolioAggregationContract` adds per-account, per-wallet, and per-group queries |
| Portfolio Aggregation → Asset Valuator | **No change** | Asset valuation is account-agnostic |
| Portfolio Aggregation → CEX Integration | **No change** | CEX integrations authenticate independently; their data is attributed at the aggregation layer |

### Data Flow — Updated

```
User connects wallet in CygnusWealthApp
    │
    ▼
WalletIntegrationSystem
    ├── Creates WalletConnection with ConnectedAccounts
    ├── Persists to encrypted local storage
    ├── Emits onWalletConnected, onAccountDiscovered
    │
    ▼
PortfolioAggregation (AccountRegistryService)
    ├── Registers TrackedAddresses from WalletIntegrationContract
    ├── Deduplicates addresses across connections
    ├── Builds AddressRequest[] for each integration
    │
    ├───► EvmIntegration.getBalances(addressRequests)
    │       ├── Groups by unique (address, chainId)
    │       ├── Batch RPC queries
    │       ├── Fans out results to all associated AccountIds
    │       └── Returns AccountBalanceList
    │
    ├───► SolIntegration.getBalances(addressRequests)
    │       └── Same pattern
    │
    ├───► CoinbaseIntegration.getBalances()
    │       └── CEX data (no address-based query needed)
    │
    ├───► AssetValuator.getBatchPrices(symbols)
    │       └── Account-agnostic pricing
    │
    ▼
PortfolioAggregation (PortfolioAggregationService)
    ├── Combines all integration results
    ├── Attributes assets to AccountIds
    ├── Deduplicates same-address holdings in aggregate
    ├── Builds Portfolio with accountBreakdown and walletBreakdown
    │
    ▼
CygnusWealthApp (CacheStore)
    ├── Stores aggregate portfolio
    ├── Stores per-account portfolios
    ├── Triggers UI re-renders via Zustand selectors
    │
    ▼
User sees portfolio with per-account attribution
```

---

## 8. Migration & Backward Compatibility

### Existing Single-Wallet Users

Users who already have addresses configured (tracked in the current flat model) need a migration path:

1. **Auto-migration on first load**: When the app detects legacy flat-model address data in local storage:
   a. Create a single `WalletConnection` with `providerId: 'manual'` and `connectionLabel: 'Imported Addresses'`
   b. Each legacy tracked address becomes a `ConnectedAccount` with `source: 'manual'`
   c. Labels and chain scopes preserved from legacy data
   d. Legacy data structure removed after migration
2. **Watch addresses**: Any addresses that were manually added (not via wallet connection) in the legacy system become `WatchAddress` entries.
3. **No data loss**: All previously tracked addresses continue to be tracked. The migration adds structure without removing anything.

### Backward Compatibility of Contracts

- `BlockchainIntegrationContract`: The old `getBalances(addresses: string[])` signature is superseded by `getBalances(requests: AddressRequest[])`. Since these are internal module interfaces (not public APIs), no backward compatibility layer is needed — the migration is a clean switch at implementation time.
- `PortfolioAggregationContract`: New methods (`getAccountPortfolio`, `getWalletPortfolio`, etc.) are additive. The existing `getPortfolio()` continues to return the aggregate view.
- `WalletIntegrationContract`: New methods are additive. The existing `getConnectedAddresses()` is superseded by `getTrackedAddresses()` but can be preserved as a convenience wrapper.

### Data Model Versioning

Add a `schemaVersion: number` field to the persisted local storage data:
- Version 1: Legacy flat address model
- Version 2: Multi-wallet multi-account model
- On load, check version and run migration if needed

---

## 9. Interaction with Existing Directives

### en-p0ha (WebSocket-First Architecture)

- **SubscriptionService**: No interface change needed. Subscriptions are address-based, not account-based. The `SubscriptionEvent` is extended with `accountIds` for attribution.
- **SubscriptionOrchestrator**: Updated to resolve `accountIds` from addresses via AccountRegistryService after receiving events.
- **LiveUpdateStore**: Updated to track `lastAccountUpdateTimestamp` per account.
- **PollManager**: No change. Polling is address-based.

### en-25w5 (RPC Provider Fallback Chain)

- **RpcProviderConfig**: No change. RPC configuration is account-agnostic.
- **Fallback chain**: No change. The number of unique addresses may increase, but the fallback chain operates per-chain, not per-account.

### en-9qon / en-v99r (Cross-Chain DeFi Architecture)

- **DeFi positions**: Already attributed by `ownerAddress`. Add `accountId` to `DeFiPosition` for direct lookup.
- **Dual-path discovery**: Runs per address, not per account. Account attribution happens at the portfolio aggregation layer after DeFi data is returned.

### hq-onegb / hq-su5pt (CEX Integrations)

- **CEX data**: CEX integrations authenticate via OAuth/API keys, not wallet connections. CEX accounts are not wallet-connected accounts — they are separate integration sources.
- **Reconciliation**: When a CEX reports holdings that also exist on-chain at a tracked address, the existing reconciliation rule (on-chain preferred) applies. The on-chain holding is attributed to the appropriate `AccountId`; the CEX duplicate is suppressed in aggregate.

---

## 10. Performance Considerations

### Scale Targets

| Metric | Target |
|--------|--------|
| Wallet connections | Up to 10 simultaneous |
| Accounts per connection | Up to 20 |
| Total tracked accounts | Up to 100 (connections + watch addresses) |
| Unique addresses (after dedup) | Up to 50 |
| Chains per address | Up to 8 |
| Total (address, chain) pairs | Up to 400 |

### Optimization Strategies

1. **Address-level deduplication**: Query each unique address once, regardless of how many AccountIds reference it.
2. **Chain scoping**: Only query addresses on chains they're configured to track.
3. **Lazy per-account portfolios**: `AccountPortfolio` objects are computed on demand (when the user views that account), not eagerly for all accounts on every refresh.
4. **Incremental updates**: Live updates (from en-p0ha WebSocket subscriptions) update only the affected account(s), not the full portfolio tree.
5. **Batch rendering**: Account portfolio updates within a 100ms window are batched into a single render cycle (existing pattern from en-p0ha).

### Memory Budget

- Each `WalletConnection` with 5 accounts: ~2KB serialized
- Each `AccountPortfolio` with 50 assets: ~15KB serialized
- 100 accounts × 15KB = ~1.5MB for full per-account portfolio cache
- Well within the existing 10MB local storage budget

---

## 11. Testing Scenarios

### WalletIntegrationSystem Tests

- Connect MetaMask, receive single account, switch accounts 3 times → 3 unique accounts accumulated
- Connect Rabby, receive 4 accounts at once → 4 accounts stored immediately
- Connect WalletConnect, user approves 2 accounts → 2 accounts stored
- Disconnect wallet → connection removed, accounts no longer tracked
- Session restore → persisted accounts loaded, stale detection works
- Same address in MetaMask and Rabby → both connections track it, distinct AccountIds
- Add manual watch address → tracked without wallet connection
- Remove watch address → no longer tracked

### PortfolioAggregation Tests

- 3 accounts across 2 wallets, 1 watch address → aggregate portfolio combines all
- Same address in 2 wallets → aggregate portfolio does not double-count
- Per-account view → only shows that account's holdings
- Per-wallet view → shows rollup of wallet's accounts
- Account group view → shows rollup of group members
- Partial failure (one chain down) → available accounts show data, failed chain shows cached/error
- Live update for an address in 2 connections → both AccountPortfolios updated

### CygnusWealthApp Tests

- View mode switching: aggregate → by-wallet → by-account → by-group
- Account filter chips: toggle accounts, verify portfolio recalculates
- Label editing: change account label, verify persistence and display
- Wallet connection flow: full connect → discover accounts → label → confirm
- Account discovery toast: switch in wallet, verify toast appears, verify account added
- Migration: legacy flat data → multi-account structure preserved

---

## 12. Implementation Sequencing

The recommended implementation order respects the dependency graph:

### Phase 1: DataModels — Identity Types
- Add all new types: `WalletProviderId`, `WalletConnectionId`, `AccountId`, `WalletConnection`, `ConnectedAccount`, `WatchAddress`, `AccountGroup`
- Add `AccountPortfolio`, `WalletPortfolio`, updated `Portfolio` fields
- Add `accountId` field to `Asset`, `Transaction`, `DeFiPosition`
- Add `AddressRequest`, `TrackedAddress`, `AccountMetadata`

### Phase 2: WalletIntegrationSystem — Multi-Account Connection
- Implement account accumulation model
- Implement watch address management
- Implement account group CRUD
- Implement session persistence with schema versioning
- Implement EIP-1193 `accountsChanged` listener with accumulation
- Update both contracts (`WalletConnectionContract`, `WalletIntegrationContract`)

### Phase 3: EvmIntegration — Account-Attributed Results
- Switch from `string[]` to `AddressRequest[]` in all service methods
- Implement address deduplication in batch queries
- Add `accountId` attribution to all result types
- Add `accountIds` to subscription events

### Phase 4: PortfolioAggregation — Account-Aware Orchestration
- Replace `Address Registry Service` with `Account Registry Service`
- Implement `resolveAccountIds` for address → account mapping
- Implement per-account, per-wallet, per-group portfolio computation
- Update deduplication rules for same-address-multiple-accounts
- Update SubscriptionOrchestrator with account resolution
- Add all new query methods to `PortfolioAggregationContract`

### Phase 5: CygnusWealthApp — Multi-Account UX
- Update `UserConfigurationStore` with wallet connection model
- Update `CacheStore` with per-account portfolio caching
- Update `LiveUpdateStore` with per-account timestamps
- Implement Wallet Management Panel
- Implement portfolio view mode selector and account filters
- Implement account discovery toast
- Implement legacy data migration

### Dependency Order

```
Phase 1 (data-models types)
    │
    ├──► Phase 2 (WalletIntegrationSystem)
    │
    └──► Phase 3 (EvmIntegration)
              │
              └──► Phase 4 (PortfolioAggregation)
                        │
                        └──► Phase 5 (CygnusWealthApp)
```

Phases 2 and 3 can proceed in parallel after Phase 1.
Phase 4 requires both Phase 2 and Phase 3.
Phase 5 requires Phase 4.

---

## Related Documentation

- **[Wallet Integration System](../domains/integration/bounded-contexts/wallet-integration-system.md)** — Current wallet connection architecture
- **[EVM Integration](../domains/integration/bounded-contexts/evm-integration.md)** — Current EVM data fetching
- **[Portfolio Aggregation](../domains/portfolio/bounded-contexts/portfolio-aggregation.md)** — Current aggregation and deduplication
- **[CygnusWealth App](../domains/experience/bounded-contexts/cygnus-wealth-app.md)** — Current UI patterns and state management
- **[Data Models](../domains/contract/bounded-contexts/data-models.md)** — Current shared types
- **[Domain Contracts](../contracts.md)** — Current inter-domain contracts
- **[WebSocket-First Architecture](./websocket-first-architecture.md)** — Live update infrastructure affected by account attribution
- **[RPC Provider Fallback Chain](./rpc-provider-fallback-chain.md)** — RPC infrastructure (unaffected)
