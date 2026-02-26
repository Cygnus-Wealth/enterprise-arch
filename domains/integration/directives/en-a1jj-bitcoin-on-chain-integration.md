# en-a1jj: Bitcoin On-Chain Integration Architecture

**Status**: RECOMMENDATION
**Date**: 2026-02-25
**Bead**: en-a1jj
**Builds On**: en-o8w (Multi-Chain Wallet Unification)
**Scope**: Integration Domain, Bitcoin on-chain bounded context, UTXO model considerations, data source strategy, wallet detection, DataModels extensions, portfolio aggregation routing

---

## Executive Summary

**Bitcoin is the first UTXO-based blockchain integration for CygnusWealth, requiring architectural patterns distinct from the existing account-based integrations (EVM, Solana).**

en-o8w established Bitcoin as a future chain family (`ChainFamily.BITCOIN`) with a routing placeholder: `chainFamily=bitcoin → btc-integration.getBalances()`. The types already exist in `data-models`: `Chain.BITCOIN`, `ChainFamily.BITCOIN`, `IntegrationSource.BITCOIN`, and the CAIP-2 `bip122` namespace mapping.

This directive defines the bounded context architecture for `btc-integration`: a read-only, client-side TypeScript library that fetches Bitcoin balances, transaction history, and UTXO sets from public block explorer APIs, transforms them into CygnusWealth's unified data models, and exposes a service facade for consumption by portfolio-aggregation. The integration follows the layered architecture established by existing integrations (Adapter → Translator → Validator → Domain Model) while addressing Bitcoin-specific concerns: the UTXO model, multiple address formats, absence of a wallet detection standard, and privacy considerations around address reuse and xpub exposure.

### Key Constraint: Read-Only, Client-Side Only

All constraints from en-o8w carry forward. We never access private keys, sign transactions, or make server calls. Bitcoin integration is purely about address balance lookups, UTXO enumeration, and transaction history via public APIs.

---

## 1. Bounded Context Definition

### Identity

- **Name**: Bitcoin Integration
- **Repository**: `btc-integration`
- **Package**: `@cygnus-wealth/btc-integration`
- **Domain**: Integration
- **GitHub Topics**: `domain-integration`, `context-btc`, `type-library`, `blockchain`, `typescript`

### Core Responsibilities

1. **Balance retrieval**: Fetch confirmed and unconfirmed BTC balances for one or more addresses
2. **UTXO enumeration**: Retrieve the unspent transaction output set for address-level portfolio detail
3. **Transaction history**: Paginated retrieval of confirmed and mempool transactions
4. **Address format support**: Accept and validate all four Bitcoin address formats (P2PKH, P2SH, Segwit bech32, Taproot bech32m)
5. **Data transformation**: Convert block explorer API responses to CygnusWealth unified data models
6. **Multi-address aggregation**: Aggregate balances across multiple addresses belonging to the same wallet connection

### Domain Boundaries

- **Owns**: Bitcoin block explorer API interactions, address format validation, UTXO-to-balance aggregation, satoshi-to-BTC conversion, Bitcoin-specific data transformation
- **Does NOT Own**: Private key management, transaction signing/broadcasting, wallet derivation paths (HD wallet key generation), Bitcoin node operation, fee estimation for sending

### Read-Only Guarantee

The integration uses only public read endpoints from block explorer APIs. No endpoints that accept signed transactions, broadcast raw transactions, or modify state are invoked. The integration never requests or handles private keys, xpubs, or derivation paths.

---

## 2. Architecture Decisions

### 2.1 UTXO Model vs Account Model

**Decision**: Model Bitcoin as a UTXO-based system internally but expose account-model-compatible interfaces to upstream consumers.

**Rationale**: Bitcoin does not have an "account balance" concept at the protocol level. A wallet's balance is the sum of all unspent transaction outputs (UTXOs) controlled by its addresses. However, CygnusWealth's unified data model is account-based (`Balance[]` on `Account`). The btc-integration must bridge this gap:

- Fetch UTXOs per address from the block explorer
- Sum UTXO values to derive an address balance
- Aggregate address balances into a single `Balance` for the wallet connection
- Optionally expose the UTXO set in metadata for advanced users

This preserves the unified portfolio model while capturing Bitcoin's native data structure.

### 2.2 Address Format Support

**Decision**: Support all four active Bitcoin mainnet address formats.

| Format | Prefix | Example Pattern | BIP |
|--------|--------|----------------|-----|
| **P2PKH** (Legacy) | `1` | `1A1zP1eP5QGefi2DMPTfTL5SLmv7DivfNa` | BIP-44 |
| **P2SH** (Nested SegWit) | `3` | `3J98t1WpEZ73CNmQviecrnyiWrnqRhWNLy` | BIP-49 |
| **Bech32** (Native SegWit) | `bc1q` | `bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4` | BIP-84 |
| **Bech32m** (Taproot) | `bc1p` | `bc1p5cyxnuxmeuwuvkwfem96lqzszee02lescdyuxqdl...` | BIP-86 |

**Rationale**: All four formats are in active use. Modern wallets (Xverse, Unisat, Leather) may expose Taproot addresses by default while legacy wallets still use P2PKH. The integration must handle all formats to avoid silently missing balances.

### 2.3 Data Source Strategy

**Decision**: Use public block explorer REST APIs as the primary data source, with a provider fallback chain.

| Provider | API Style | Rate Limits | Coverage | Priority |
|----------|-----------|------------|----------|----------|
| **Mempool.space** | REST + WebSocket | Generous (public instance) | Mainnet, testnet, signet | Primary |
| **Blockchair** | REST | 30 req/min (free tier) | Mainnet + many chains | Secondary |
| **Esplora (Blockstream.info)** | REST | Moderate | Mainnet, testnet | Tertiary |

**Rationale**: A self-hosted Electrum server or Bitcoin full node is not viable for a client-side, browser-only application. Public block explorer APIs provide the needed read-only data (balances, UTXOs, transactions) without requiring infrastructure. Mempool.space is the primary choice because it is open-source, well-documented, widely used, and has the Esplora-compatible API format that Blockstream.info also uses, enabling easy fallback.

**Self-hosted option**: If CygnusWealth later deploys server-side infrastructure, a self-hosted Mempool.space or Esplora instance can slot into the same adapter interface. This is a future enhancement, not a current requirement.

### 2.4 Scope: Native BTC Only

**Decision**: Scope to native BTC balances and transactions only. Ordinals, BRC-20 tokens, and Runes are out of scope for this directive.

**Rationale**: Ordinals, BRC-20, and Runes are Bitcoin meta-protocols built on top of the base UTXO layer. They require specialized indexing (e.g., `ord` server for Ordinals, custom BRC-20 indexers) that is not available from standard block explorer APIs. Adding them would significantly increase complexity, require additional data sources, and introduce unstable/evolving protocol dependencies.

**Future work**: A separate directive can define Ordinals/BRC-20/Runes support as an extension to this bounded context, once the ecosystem and indexing infrastructure stabilize.

### 2.5 Wallet Detection Approach

**Decision**: Use wallet-specific global injection detection, as mandated by en-o8w for Bitcoin wallets (no finalized standard exists).

**Known Bitcoin wallet injection points**:

| Wallet | Global Object | Multi-Chain | Notes |
|--------|--------------|-------------|-------|
| **Xverse** | `window.XverseProviders.BitcoinProvider` | Yes (BTC + STX) | Also registers via Wallet Standard for STX |
| **Unisat** | `window.unisat` | No (BTC only) | Exposes `getAccounts()`, `getBalance()` |
| **Leather (Hiro)** | `window.LeatherProvider` or `window.HiroWalletProvider` | Yes (BTC + STX) | Formerly Hiro Wallet |
| **Phantom** | `window.phantom.bitcoin` | Yes (EVM + SOL + BTC) | Detected via EIP-6963/Wallet Standard for other chains; BTC sub-provider at `phantom.bitcoin` |
| **OKX Wallet** | `window.okxwallet.bitcoin` | Yes (multi-chain) | BTC sub-provider under okxwallet namespace |

**Correlation with en-o8w**: For multi-chain wallets (Phantom, Xverse), the WalletIntegration domain correlates the Bitcoin provider detection with EIP-6963/Wallet Standard providers from the same wallet. This produces a unified `DiscoveredWallet` entry. Domain Arch for WalletIntegration determines the specific correlation heuristics.

**Fallback**: If no Bitcoin-specific injection is detected, users can manually add Bitcoin addresses as watch-only. This is a common pattern since many users hold BTC in hardware wallets (Ledger, Trezor, Coldcard) that don't inject browser providers.

### 2.6 Single-Address Model (No xpub/HD Wallet Derivation)

**Decision**: The integration accepts individual Bitcoin addresses, not xpubs or derivation paths.

**Rationale**: Exposing an xpub allows anyone to derive all past and future addresses in the wallet, which is a significant privacy concern. CygnusWealth's read-only, client-side model means the user provides addresses explicitly (via wallet injection or manual entry). The integration tracks exactly the addresses it is given — it does not derive additional addresses from HD wallet paths.

**Implication**: If a user's wallet has spread funds across many derived addresses, only the addresses explicitly provided will be tracked. This is an acceptable trade-off for privacy. The UX layer can inform users about this limitation and suggest adding all active addresses.

---

## 3. Data Model Mappings

### Mapping to Existing DataModels Types

#### Balance Mapping

| Bitcoin Concept | DataModels Type | Field Mapping |
|----------------|----------------|---------------|
| Sum of UTXO values per address | `Balance` | `amount`: total in BTC (string, 8 decimal places) |
| BTC asset | `Asset` | `symbol: 'BTC'`, `name: 'Bitcoin'`, `decimals: 8` |
| Confirmed + unconfirmed | `Balance.metadata` | `"btc:confirmedBalance"`, `"btc:unconfirmedBalance"` |
| UTXO count | `Balance.metadata` | `"btc:utxoCount"` |

Balances are presented in BTC (not satoshis) for consistency with the unified portfolio display. Internal calculations use satoshi integers to avoid floating-point precision issues, converting to BTC strings at the mapping boundary.

#### Transaction Mapping

| Bitcoin Concept | DataModels `Transaction` Field | Notes |
|----------------|-------------------------------|-------|
| Transaction ID (txid) | `id` | 64-character hex string |
| Block height | `metadata["btc:blockHeight"]` | `null` for unconfirmed |
| Block timestamp | `timestamp` | Unix timestamp → ISO string; mempool txs use `Date.now()` |
| Confirmations | `metadata["btc:confirmations"]` | 0 for mempool |
| Total inputs from tracked addresses | `assetsOut[0].amount` | BTC sent (if any) |
| Total outputs to tracked addresses | `assetsIn[0].amount` | BTC received (if any) |
| Fee | `fees[0].amount` | In BTC; only present when sending |
| Transaction type | `type` | See type determination below |

**Transaction type determination**: Bitcoin transactions don't have inherent types. The integration determines type based on the tracked addresses' involvement:

| Scenario | `TransactionType` |
|----------|-------------------|
| Tracked address appears only in outputs (receiving) | `TRANSFER_IN` |
| Tracked address appears only in inputs (sending) | `TRANSFER_OUT` |
| Tracked address in both inputs and outputs (self-transfer/change) | `TRANSFER_OUT` (net amount after change) |

#### UTXO Mapping (Metadata Extension)

UTXO details are exposed via the metadata namespace pattern, not as new top-level types:

```
"btc:utxos": [
  { "txid": "...", "vout": 0, "value": "0.05000000", "confirmations": 6 },
  ...
]
```

This follows the established metadata extension pattern from Kraken (e.g., `"kraken:holdTrade"`).

### IntegrationSource — Already Present

`IntegrationSource.BITCOIN` is already defined in the DataModels enum per en-o8w. No change needed.

### ChainFamily — Already Present

`ChainFamily.BITCOIN` is already defined per en-o8w. The CAIP-2 namespace `bip122` maps to `ChainFamily.BITCOIN`.

---

## 4. New Types Needed in DataModels

### Assessment: No New Top-Level Types Required

Bitcoin integration maps cleanly to existing DataModels types via the metadata extension pattern:

- Address balances → `Balance[]` with `asset: BTC`
- Transaction history → `Transaction[]` with appropriate `TransactionType`
- UTXOs → `Balance.metadata["btc:utxos"]`
- Confirmation status → `Transaction.metadata["btc:confirmations"]`

### Metadata Namespace Convention for Bitcoin

All Bitcoin-specific data stored in metadata fields follows the existing namespace convention:

```
"btc:confirmedBalance"     // Confirmed balance in BTC
"btc:unconfirmedBalance"   // Unconfirmed (mempool) balance in BTC
"btc:utxoCount"            // Number of unspent outputs
"btc:utxos"                // UTXO array (optional, for advanced display)
"btc:blockHeight"          // Block height of transaction
"btc:confirmations"        // Number of confirmations
"btc:addressFormat"        // "p2pkh" | "p2sh" | "bech32" | "bech32m"
"btc:fee"                  // Transaction fee in BTC
"btc:size"                 // Transaction size in vbytes
"btc:isCoinbase"           // Whether transaction is a coinbase (mining) reward
```

No new interface needed — this uses the existing `Metadata` extension object pattern (`{[key: string]: unknown}`).

### Address Validation Types

Bitcoin address format validation is internal to the btc-integration bounded context. It does NOT belong in shared DataModels because:
- Address format is a Bitcoin-specific implementation concern
- Other domains never need to validate Bitcoin address formats
- The integration accepts addresses as opaque strings at its boundary

Domain Arch for btc-integration defines the internal validation types.

---

## 5. Rig Structure

### Package Layout

Following the established pattern from existing integrations:

```
btc-integration/
├── refinery/rig/
│   ├── src/
│   │   ├── api/
│   │   │   ├── client.ts              // HTTP transport, provider fallback
│   │   │   ├── endpoints.ts           // Endpoint path constants per provider
│   │   │   └── btc-api.ts            // Domain operation → endpoint mapping
│   │   ├── types/
│   │   │   └── index.ts              // Block explorer API response types (raw shapes)
│   │   ├── models/
│   │   │   └── standardized.ts       // Re-exports from @cygnus-wealth/data-models
│   │   ├── mappers/
│   │   │   ├── balance-mapper.ts     // UTXO aggregation → Balance[]
│   │   │   ├── transaction-mapper.ts // Bitcoin txs → Transaction[]
│   │   │   ├── utxo-mapper.ts        // Raw UTXOs → metadata format
│   │   │   └── address-validator.ts  // Address format detection and validation
│   │   ├── services/
│   │   │   └── btc-service.ts        // Facade: API + mappers orchestration
│   │   ├── index.ts                  // Public API exports
│   │   └── __tests__/
│   │       ├── client.test.ts
│   │       ├── btc-api.test.ts
│   │       ├── btc-service.test.ts
│   │       ├── balance-mapper.test.ts
│   │       ├── transaction-mapper.test.ts
│   │       ├── utxo-mapper.test.ts
│   │       ├── address-validator.test.ts
│   │       └── e2e/
│   │           ├── fixtures/
│   │           │   └── btc-responses.json
│   │           └── btc-e2e.test.ts
│   ├── package.json
│   ├── tsconfig.json
│   ├── vitest.config.ts
│   ├── vitest.e2e.config.ts
│   ├── ARCHITECTURE.md
│   └── README.md
├── mayor/rig/                         // Replicated instance (same structure)
├── witness/
├── plugins/
└── config.json
```

### Layered Architecture

```
┌─────────────────────────────────────────────────────┐
│                   BtcService                         │
│          (Facade — public API surface)               │
│  getBalances() getTransactions() getUTXOs()          │
│  validateAddress() getAddressInfo()                  │
└──────────────────┬──────────────────┬───────────────┘
                   │                  │
           ┌───────▼────────┐  ┌─────▼──────────┐
           │     BtcAPI      │  │    Mappers      │
           │ (domain ops →   │  │ (Bitcoin →      │
           │  endpoint map)  │  │  DataModels)    │
           └───────┬─────────┘  └────────────────┘
                   │
           ┌───────▼─────────┐
           │   BtcClient     │
           │ (HTTP transport, │
           │  provider        │
           │  fallback chain) │
           └─────────────────┘
```

### Layer Responsibilities

#### BtcClient (api/client.ts)

HTTP transport with provider fallback:
- Wraps `fetch` (not axios — lighter for browser bundles)
- Implements provider fallback chain: Mempool.space → Blockchair → Esplora
- Detects provider failures and switches to next provider (circuit breaker per provider)
- Handles provider-specific response formats
- Configurable timeout (default 15s), retry (default 3 attempts)

#### BtcAPI (api/btc-api.ts)

Maps domain operations to raw API calls:
- `getAddressBalance(address)` → provider-specific balance endpoint
- `getAddressUTXOs(address)` → provider-specific UTXO endpoint
- `getAddressTransactions(address, afterTxid?)` → provider-specific tx history endpoint
- `getTransaction(txid)` → provider-specific tx detail endpoint
- `getBlockTipHeight()` → current block height (for confirmation calculation)

#### Mappers (mappers/*.ts)

Pure transformation functions (static methods, no state, no I/O):

- **BalanceMapper**: Aggregates UTXOs into `Balance[]`. Separates confirmed vs unconfirmed. Converts satoshis to BTC strings with 8 decimal precision.
- **TransactionMapper**: Maps raw Bitcoin transactions to `Transaction[]`. Determines direction (in/out/self) by comparing inputs/outputs against tracked addresses. Calculates net transfer amounts.
- **UTXOMapper**: Transforms raw UTXO arrays into the metadata format for optional detailed display.
- **AddressValidator**: Validates Bitcoin address formats (P2PKH, P2SH, bech32, bech32m) and detects format type. Uses bech32/bech32m decoding for SegWit addresses, base58check for legacy.

#### BtcService (services/btc-service.ts)

Facade orchestrating all layers:

- **`getBalances(addresses)`**: Fetches UTXOs for each address, aggregates via BalanceMapper, returns combined `Balance[]`
- **`getTransactions(addresses, options?)`**: Paginates through transaction history, maps via TransactionMapper, returns `Transaction[]` with direction determination
- **`getUTXOs(addresses)`**: Raw UTXO set for all addresses (for metadata enrichment)
- **`validateAddress(address)`**: Returns validity and format type
- **`getAddressInfo(address)`**: Returns balance + tx count + UTXO count (lightweight summary)

Error wrapping: All methods wrap errors in `StandardizedError` with `source: 'bitcoin'`.

### Public API Exports

```
BtcService                 // Main service class (primary public API)
BtcClient                  // Low-level HTTP client (for advanced use)
BtcAPI                     // API operation mapper
BalanceMapper              // Balance transformation
TransactionMapper          // Transaction transformation
UTXOMapper                 // UTXO transformation
AddressValidator           // Address format validation
* from './types'           // All block explorer API response types
```

---

## 6. API Strategy

### Mempool.space (Primary Provider)

Uses the Esplora-compatible API format:

| Endpoint | Path | Purpose | CygnusWealth Mapping |
|----------|------|---------|---------------------|
| Address info | `GET /api/address/{address}` | Balance summary (funded/spent/total) | `Balance` (computed from chain stats) |
| Address UTXOs | `GET /api/address/{address}/utxo` | Unspent outputs | `Balance.metadata["btc:utxos"]` |
| Address txs | `GET /api/address/{address}/txs` | Transaction history (25 per page) | `Transaction[]` |
| Address txs (paginated) | `GET /api/address/{address}/txs/chain/{last_seen_txid}` | Paginated tx history | `Transaction[]` (continued) |
| Transaction detail | `GET /api/tx/{txid}` | Full transaction with inputs/outputs | `Transaction` enrichment |
| Block tip height | `GET /api/blocks/tip/height` | Current block height | Confirmation calculation |

**Base URL**: `https://mempool.space` (public instance) or user-configured self-hosted instance.

### Blockchair (Secondary Provider)

| Endpoint | Path | Purpose | Notes |
|----------|------|---------|-------|
| Address stats | `GET /bitcoin/dashboards/address/{address}` | Balance, tx count, UTXOs | Different response format from Esplora |
| Address txs | `GET /bitcoin/dashboards/address/{address}?transaction_details=true&limit=100` | Transactions with details | Max 100 per call |
| Transaction | `GET /bitcoin/dashboards/transaction/{txid}` | Full tx detail | Separate response format |

**Rate limit**: 30 requests/minute on free tier. The adapter implements request pacing.

### Esplora / Blockstream.info (Tertiary Provider)

Same API format as Mempool.space (both implement the Esplora API specification). Only the base URL differs: `https://blockstream.info`.

### Pagination Strategy

Mempool.space/Esplora uses cursor-based pagination for transaction history:
- First call: `GET /api/address/{address}/txs` (returns 25 most recent)
- Subsequent calls: `GET /api/address/{address}/txs/chain/{last_seen_txid}` (returns next 25 after the given txid)
- Continue until response contains fewer than 25 transactions

Blockchair uses offset/limit parameters when used as fallback.

---

## 7. UTXO Model Considerations

### Why Bitcoin is Different

EVM and Solana integrations query a single balance endpoint that returns the current account state. Bitcoin has no such concept:

- **No global state**: There is no "account balance" stored anywhere. Balance is computed from the UTXO set.
- **Multiple addresses**: A single wallet typically uses many addresses (change addresses, different derivation paths).
- **Unconfirmed transactions**: Mempool transactions create unconfirmed UTXOs that affect the "pending" balance.
- **UTXO fragmentation**: A wallet may hold hundreds of small UTXOs from multiple transactions, affecting both balance accuracy and display.

### Aggregation Rules

1. **Per-address**: Fetch UTXOs for each tracked address independently
2. **Sum confirmed**: Aggregate confirmed UTXO values across all addresses → `Balance.amount`
3. **Sum unconfirmed separately**: Unconfirmed UTXOs → `Balance.metadata["btc:unconfirmedBalance"]`
4. **Total = confirmed + unconfirmed**: The `Balance.amount` field reflects the total (confirmed + unconfirmed), with breakdown available in metadata
5. **No cross-address deduplication**: UTXOs are unique by `(txid, vout)`, so no deduplication is needed

### Performance Implications

A user with a high-transaction-volume wallet may have thousands of UTXOs across multiple addresses. Mitigations:

- **Parallel address fetching**: Fetch UTXOs for all addresses concurrently
- **UTXO summary mode**: For balance display, only the aggregated value is needed (not the full UTXO list)
- **Lazy UTXO loading**: Full UTXO details loaded only on demand (e.g., when user expands address detail)
- **Caching**: UTXO sets for confirmed transactions are stable and cacheable; only mempool state changes frequently

---

## 8. Resilience Patterns

Applying the Integration Domain patterns from `patterns.md`:

### Circuit Breaker Configuration

Per-provider circuit breakers (independent for Mempool.space, Blockchair, Esplora):

```
failureThreshold: 5
successThreshold: 3
timeout: 15000ms
volumeThreshold: 10
errorFilter: Only count 5xx, network, and rate limit errors (not 404s for unknown addresses)
```

When a provider's circuit opens, requests automatically route to the next provider in the fallback chain.

### Retry Configuration

```
maxAttempts: 3
baseDelay: 1000ms
maxDelay: 15000ms
multiplier: 2
jitterFactor: 0.3
retryableErrors: [5xx, network errors, rate limit (429)]
nonRetryableErrors: [400 (invalid address), 404 (address not found)]
```

### Caching Strategy

| Data Type | TTL | Cache Layer | Rationale |
|-----------|-----|-------------|-----------|
| Address balance (confirmed) | 60 seconds | L1 (memory) | New blocks every ~10 minutes; 60s is responsive enough |
| Address UTXOs | 60 seconds | L1 (memory) | Changes with each new block or mempool update |
| Transaction history (confirmed) | 10 minutes | L1 + L2 (IndexedDB) | Confirmed txs are immutable; only new txs need fetching |
| Transaction detail | 24 hours | L2 (IndexedDB) | Confirmed tx data never changes |
| Block tip height | 30 seconds | L1 (memory) | Used for confirmation count; changes every ~10 minutes |
| Address format validation | Permanent | L1 (memory) | Deterministic — format doesn't change |

### Graceful Degradation

- If all providers fail, return last cached balances with staleness indicator
- If UTXO endpoint fails but balance endpoint succeeds, return balance without UTXO detail
- If transaction history fails, return balance-only portfolio entry
- If mempool data is unavailable, return confirmed-only balances

---

## 9. Integration with Portfolio Aggregation

### Contract Compliance

The `btc-integration` package implements the `BlockchainIntegrationContract` pattern, making it pluggable into `portfolio-aggregation`:

- `getBalances(addresses)` returns `Balance[]` with `asset: BTC`
- `getTransactions(addresses, options)` returns `Transaction[]` using the unified asset flow model (assetsIn/assetsOut/fees)
- All types are from `@cygnus-wealth/data-models`

### Registration

Portfolio-aggregation's Integration Registry registers Bitcoin as a blockchain integration:

- **Integration ID**: `bitcoin`
- **Source**: `IntegrationSource.BITCOIN`
- **Chain Family**: `ChainFamily.BITCOIN`
- **Capabilities**: `['balance', 'transactions']`
- **Refresh Strategy**: Poll-based (block explorer polling at configurable interval, default 60s)

### Routing (from en-o8w)

```
PortfolioAggregation receives TrackedAddress[]
    │
    ├── Group by chainFamily
    │
    ├── chainFamily=evm → evm-integration.getBalances()
    ├── chainFamily=solana → sol-integration.getBalances()
    ├── chainFamily=bitcoin → btc-integration.getBalances()  ← NEW
    │
    ├── Combine results, attribute to AccountIds
    │
    ▼
    Portfolio with multi-chain, multi-account attribution
```

### Data Priority

From existing architecture: "Prefer on-chain data over CEX data." Bitcoin on-chain balances are the authoritative source. If the same BTC appears in both an on-chain wallet and a CEX integration (Coinbase, Kraken), the on-chain data takes priority for deduplication.

---

## 10. Wallet Detection and Connection Flow

### Detection (WalletIntegration Domain)

Per en-o8w, Bitcoin wallet detection uses wallet-specific global injection probing. The WalletIntegration domain:

1. Probes known injection points (`window.unisat`, `window.phantom.bitcoin`, `window.XverseProviders.BitcoinProvider`, etc.)
2. Correlates Bitcoin providers with EIP-6963/Wallet Standard providers from the same wallet (for multi-chain wallets like Phantom)
3. Produces `DiscoveredWallet` entries with `supportedChainFamilies` including `'bitcoin'`

### Connection Flow

1. User selects a discovered wallet with Bitcoin support
2. WalletIntegration calls the wallet's `getAccounts()` (or equivalent) to retrieve Bitcoin addresses
3. Addresses are stored as `ConnectedAccount` entries with `chainFamily: 'bitcoin'`
4. PortfolioAggregation receives `TrackedAddress[]` with `chainFamily: 'bitcoin'` and routes to btc-integration

### Watch-Only Addresses

Users can manually add Bitcoin addresses as watch-only (no wallet connection required). This is the primary path for hardware wallet users. The address validator confirms the format before adding.

### AccountId Format

Per en-o8w: `{walletConnectionId}:bitcoin:{address}`

Example: `conn_abc123:bitcoin:bc1qw508d6qejxtdg4y5r3zarvary0c5xw7kv8f3t4`

---

## 11. Privacy Considerations

### Address Reuse

Bitcoin best practice is to use a new address for each transaction (change addresses). Tracking multiple addresses for the same wallet connection is expected and supported. The integration does not assume a 1:1 mapping between wallet connections and addresses.

### xpub Exposure

**The integration never requests, stores, or derives from xpubs.** An xpub (extended public key) can derive all past and future addresses in an HD wallet. Exposing it to any third party (including block explorer APIs) allows full transaction graph reconstruction. CygnusWealth avoids this by accepting only individual addresses.

### Block Explorer Privacy

**Risk**: Querying a block explorer API with multiple addresses reveals that those addresses are likely controlled by the same entity.

**Mitigations**:
- Each address is queried independently (no batch endpoint that reveals address groupings)
- Requests are made directly from the user's browser (no CygnusWealth server sees the addresses)
- Users who want maximum privacy can use a self-hosted Mempool.space or Esplora instance (configurable base URL)
- No address data leaves the client — no analytics, no telemetry, no server-side logging

### CAIP-2 Chain Identification

Bitcoin mainnet CAIP-2 identifier: `bip122:000000000019d6689c085ae165831e93` (genesis block hash).

---

## 12. Error Handling

### Block Explorer Error Mapping

| Error Condition | StandardizedError Code | Retriable | Action |
|----------------|----------------------|-----------|--------|
| Invalid address format | `INVALID_ADDRESS` | No | Reject with format guidance |
| Address not found (no history) | `ADDRESS_EMPTY` | No | Return zero balance (not an error) |
| Rate limited (429) | `RATE_LIMITED` | Yes | Switch to next provider or wait |
| Provider unavailable (5xx) | `SERVICE_UNAVAILABLE` | Yes | Circuit breaker + fallback |
| Network error | `NETWORK_ERROR` | Yes | Retry with backoff |
| Response parse failure | `PARSE_ERROR` | No | Log and try next provider |
| All providers failed | `ALL_PROVIDERS_FAILED` | No | Return cached data with staleness indicator |

---

## 13. Testing Strategy

### Test Distribution

Following the Integration Domain testing pyramid:

- **Unit Tests (80%)**: Address validator, mappers (balance, transaction, UTXO), client (mocked fetch), service (mocked API)
- **Integration Tests (15%)**: Full stack with mocked HTTP responses (recorded fixtures from real providers)
- **Contract Tests (4%)**: Verify all mapper outputs conform to DataModels interfaces
- **E2E Tests (1%)**: Critical path — validate address + fetch balance + verify data shape

### Critical Test Scenarios

1. **Address format validation**: All four formats accepted; invalid formats rejected
2. **UTXO aggregation**: Multiple UTXOs across multiple addresses sum correctly
3. **Satoshi precision**: Satoshi-to-BTC conversion preserves 8 decimal places without floating-point errors
4. **Transaction direction**: Correctly determines TRANSFER_IN, TRANSFER_OUT, and self-transfer
5. **Provider fallback**: Primary failure → secondary → tertiary, with circuit breaker state management
6. **Zero-balance addresses**: Return valid `Balance` with `amount: "0"`, not an error
7. **Mempool transactions**: Unconfirmed transactions appear with `confirmations: 0`
8. **Pagination completeness**: Address with 100+ transactions fully paginated
9. **Cache behavior**: Confirmed tx details served from cache; balances refreshed on TTL expiry
10. **Privacy**: No batch requests that group addresses; each address queried independently

---

## 14. Implementation Sequence

1. **AddressValidator** — Bitcoin address format detection and validation for all four formats + unit tests
2. **BtcClient** — HTTP transport with provider fallback chain (Mempool.space → Blockchair → Esplora)
3. **Bitcoin raw types** — TypeScript interfaces for all block explorer response shapes
4. **BtcAPI** — Endpoint mapping layer with provider-specific path resolution
5. **BalanceMapper + TransactionMapper + UTXOMapper** — Data transformations with satoshi precision
6. **BtcService** — Facade orchestration with error wrapping and graceful degradation
7. **E2E tests with recorded fixtures** from real Mempool.space/Esplora responses
8. **Portfolio-aggregation integration** — Register Bitcoin as a new blockchain integration source, wire up routing

---

## Related Documentation

- **[Integration Domain README](../README.md)** — Domain overview and strategic guidance
- **[Integration Patterns](../patterns.md)** — Anti-corruption layer, circuit breaker, retry, caching patterns
- **[Resilience & Performance](../resilience-performance.md)** — Connection management and fallback strategies
- **[Testing & Security](../testing-security.md)** — Testing pyramid and security requirements
- **[en-o8w: Multi-Chain Wallet Unification](./en-o8w-multi-chain-wallet-unification.md)** — ChainFamily.BITCOIN definition, wallet detection mandate, routing placeholder
- **[hq-onegb: Coinbase CEX Integration](./hq-onegb-coinbase-cex-integration.md)** — Reference directive format
- **[hq-su5pt: Kraken CEX Integration](./hq-su5pt-kraken-cex-integration.md)** — Reference directive format
- **[Data Models](../../contract/bounded-contexts/data-models.md)** — Unified type definitions
- **[Repository Organization](../../repository-organization.md)** — GitHub repo setup and topic tagging
- **[Domain Contracts](../../contracts.md)** — Inter-domain communication contracts
