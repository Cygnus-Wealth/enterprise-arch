# en-o8w: Multi-Chain Wallet Unification

**Status**: RECOMMENDATION
**Date**: 2026-02-25
**Bead**: en-o8w
**Builds On**: en-fr0z (Multi-Wallet Multi-Account Architecture)
**Scope**: Cross-domain architecture for extending the wallet identity model from en-fr0z to support wallets spanning multiple chain families. Defines dictated discovery standards, domain ownership boundaries, shared ubiquitous language, and cross-domain contract amendments.

---

## Executive Summary

**A single wallet like Trust Wallet or Phantom exposes addresses on multiple chain families. The system must treat these as one unified wallet entity, not as unrelated connections.**

en-fr0z established the multi-wallet multi-account identity model: `WalletConnection → ConnectedAccount[]`. That directive is EVM-focused. When a user connects Trust Wallet, the system currently sees only the EVM side. The Solana, Cosmos, and Aptos sides either appear as separate unrelated connections or are invisible.

This directive extends en-fr0z across domain boundaries to unify multi-chain wallets. It defines:
- **Dictated technologies** for provider discovery across chain families
- **Domain ownership** of discovery, connection, and routing responsibilities
- **Shared ubiquitous language** introducing `ChainFamily` as a cross-domain concept
- **Cross-domain contract amendments** between WalletIntegration, PortfolioAggregation, and DataModels

### Key Constraint: Read-Only, Client-Side Only

All constraints from en-fr0z carry forward. We never access private keys, sign transactions, or make server calls. Multi-chain unification is purely about provider discovery, address correlation, and identity modeling.

---

## 1. The Multi-Chain Problem

A single wallet like Phantom injects separate chain-specific sub-providers (one for EVM, one for Solana, one for Bitcoin). There is no universal cross-chain provider API. Without unification:

1. User connects Phantom. System creates an EVM connection. Separately, user connects Phantom for Solana. System creates a second, unrelated connection. The user sees "Phantom" twice.
2. User connects Trust Wallet. Only the EVM side is detected. Solana, Cosmos, and Aptos addresses from the same wallet are invisible.
3. User connects via WalletConnect v2 and approves EVM + Solana chains in one session. The system creates two unlinked connections.

**The goal:** A single Trust Wallet connection surfaces EVM, Solana, Cosmos, and Aptos addresses — all under **one** `WalletConnection` entity.

---

## 2. Dictated Technologies

Enterprise Architecture mandates the following discovery and connection standards. Domains implementing wallet support MUST use these protocols:

| Chain Family | Dictated Standard | Rationale |
|-------------|-------------------|-----------|
| **EVM** | **EIP-6963** (Multi Injected Provider Discovery) | Replaces unreliable `window.ethereum` global race. Event-based, each wallet announces independently with UUID, name, icon, RDNS. |
| **Solana, Sui, Aptos** | **Wallet Standard** (`@wallet-standard/base`) | Registration-based discovery. Wallets register via standard events, expose chain-specific features. |
| **Remote wallets** | **WalletConnect v2** with multi-namespace sessions | Session-based discovery. CAIP-2 chain identification. Supports multiple chain families in a single session. |
| **Bitcoin, Cosmos** | Wallet-specific global injection (no finalized standard) | No industry standard yet. Domain Arch determines detection approach per wallet. |

**Fallback policy:** `window.ethereum` detection remains as fallback for EVM wallets that haven't adopted EIP-6963. `window.solana` detection remains as fallback for Solana wallets that haven't adopted Wallet Standard. Domains decide fallback implementation details.

---

## 3. Ubiquitous Language

The following terms are shared across all domains and MUST be used consistently:

### ChainFamily

A classification of blockchain protocol families. Values: `'evm'`, `'solana'`, `'sui'`, `'bitcoin'`, `'cosmos'`, `'aptos'`.

- `ChainFamily` is distinct from `ChainId` (which identifies a specific chain within a family, e.g., `ethereum`, `polygon`, `solana-mainnet`)
- `ChainFamily` is a closed enum — adding new families is an architectural decision requiring Enterprise Arch approval, a new integration bounded context, and correlation registry entries
- Used for routing (which integration handles which addresses), discovery (which standard detects which providers), and display (chain family badges in UI)

### DiscoveredWallet

A wallet detected during provider discovery, before the user has connected it. Contains:
- A canonical `WalletProviderId` (e.g., `'phantom'`, `'trust-wallet'`)
- Which `ChainFamily` values the wallet supports
- Whether the wallet is multi-chain (`isMultiChain: true` if > 1 chain family)

Discovery correlation (matching EIP-6963 EVM provider with Wallet Standard Solana provider from the same wallet) is owned by the WalletIntegration domain.

### Multi-Chain WalletConnection

An extension of en-fr0z's `WalletConnection` that spans multiple chain families. One `WalletConnection` per wallet (not per chain family). Users think of Phantom as ONE wallet — the data model reflects this.

### CAIP-2 Chain Identification

WalletConnect v2 uses CAIP-2 identifiers (e.g., `eip155:1` for Ethereum mainnet, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` for Solana mainnet). The system maintains a mapping between CAIP-2 identifiers and internal `ChainId` values. CAIP-2 namespace prefixes map to `ChainFamily` values (e.g., `eip155` → `evm`, `solana` → `solana`, `bip122` → `bitcoin`).

---

## 4. Domain Boundary Definitions

### WalletIntegration Domain — Owns Discovery and Connection

The WalletIntegration domain is solely responsible for:
- **Provider discovery**: Listening for EIP-6963 announcements, Wallet Standard registrations, and probing global injection points
- **Provider correlation**: Matching providers from the same wallet across different discovery protocols (e.g., Phantom's EIP-6963 EVM provider + Wallet Standard Solana provider → one `DiscoveredWallet`)
- **Connection management**: Per-chain-family connect/disconnect lifecycle
- **WalletConnect v2 sessions**: Session creation, namespace negotiation, session events
- **Event emission**: Notifying downstream domains of wallet/account changes

WalletIntegration does NOT fetch balances, prices, or any on-chain data.

### PortfolioAggregation Domain — Owns Routing

The PortfolioAggregation domain is solely responsible for:
- **Chain-family routing**: Receiving `TrackedAddress[]` from WalletIntegration, grouping by `chainFamily`, and dispatching to the correct blockchain integration
- **Cross-chain-family deduplication**: Addresses on different chain families are NEVER duplicates. Deduplication only applies within the same chain family (en-fr0z rules)
- **Result aggregation**: Combining results from all chain-family integrations into a unified portfolio with account/wallet attribution

### Integration Domain — Bounded Contexts per Chain Family

Each chain family has (or will have) its own integration bounded context:
- `evm-integration` — EVM chains (existing)
- `sol-integration` — Solana (existing)
- Future: `sui-integration`, `btc-integration`, `cosmos-integration`

All integration bounded contexts MUST implement the `BlockchainIntegrationContract` pattern. Each accepts `AddressRequest[]` and returns results with `accountId` attribution.

### DataModels (Contract Domain) — Shared Types

The `data-models` package owns all shared type definitions that cross domain boundaries. New shared types introduced by this directive live in `data-models`.

### Experience Domain (CygnusWealthApp) — Presentation

The Experience domain consumes discovery results and portfolio data. It presents multi-chain wallets with chain family badges and per-chain-family connection controls. Internal UX patterns are the Experience domain's concern.

---

## 5. Cross-Domain Data Model Changes (data-models)

The following types are added or extended in the shared `data-models` package:

### New Shared Types

**`ChainFamily`** — enum classifying blockchain families:
- Values: `'evm'`, `'solana'`, `'sui'`, `'bitcoin'`, `'cosmos'`, `'aptos'`

**`Caip2ChainId`** — CAIP-2 chain identifier string type:
- Format: `{namespace}:{reference}` (e.g., `eip155:1`, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`)
- Used for WalletConnect v2 interoperability

### Extended Shared Types

**`WalletConnection`** — add chain family awareness:
- New field: `supportedChainFamilies: ChainFamily[]`
- `supportedChains` becomes computed from chain providers across families

**`ConnectedAccount`** — add chain family:
- New field: `chainFamily: ChainFamily`

**`AccountId`** — format extended for chain-family namespacing:
- New format: `{walletConnectionId}:{chainFamily}:{address}`
- Existing EVM-only AccountIds gain the `:evm:` segment during migration

**`WatchAddress`** — add chain family:
- New field: `chainFamily: ChainFamily`

**`IntegrationSource`** — extended:
- Add: `SUI`, `BITCOIN`, `COSMOS`, `APTOS` (alongside existing `EVM`, `SOLANA`)

### What Does NOT Change

- `Balance`, `Value`, `MarketData`, `PriceHistory` — chain-family-agnostic
- `Portfolio`, `AccountPortfolio`, `WalletPortfolio` — already account-attributed, chain family inherits from accounts
- `Transaction` — already has `chainId`; `chainFamily` is derivable
- `ChainRpcConfig`, `RpcProviderConfig` — infrastructure-level, unchanged

---

## 6. Cross-Domain Contract Amendments

> **Scope note:** This section defines WHAT must cross domain boundaries — the required capabilities and data. Specific API names, method signatures, and event naming are Domain Arch concerns. Each domain designs its own interface to fulfill these requirements.

### App → WalletIntegration (Extended)

Building on en-fr0z, the App → WalletIntegration boundary must support these new cross-domain capabilities:

**New command capabilities:**
- **Connect with chain family selection** — App must be able to initiate a wallet connection specifying which `ChainFamily` values to connect (default: all supported by the wallet)
- **Per-chain-family connect/disconnect** — App must be able to add or remove individual chain families from an existing connection without disconnecting the entire wallet
- **WalletConnect v2 session initiation** — App must be able to start a WalletConnect v2 session specifying required and optional chain families

**New query capabilities:**
- **Discovered wallet enumeration** — App must be able to retrieve the list of `DiscoveredWallet` entries (replaces en-fr0z's provider info with chain-family-aware discovery results)
- **Wallet capability inquiry** — App must be able to query which `ChainFamily` values a specific wallet supports

**New event capabilities:**
- **Discovery completion** — WalletIntegration must notify the App when provider discovery completes with the correlated multi-chain wallet list
- **Chain family connection change** — WalletIntegration must notify the App when a chain family is added to or removed from an existing connection

Existing en-fr0z event contracts (`onWalletConnected`, `onAccountDiscovered`) continue unchanged.

### PortfolioAggregation → WalletIntegration (Extended)

Building on en-fr0z, the PortfolioAggregation → WalletIntegration boundary requires:

- **Chain-family-aware tracked addresses** — `TrackedAddress` data crossing this boundary must include `chainFamily: ChainFamily`
- **Chain-family filtering** — PortfolioAggregation must be able to query tracked addresses filtered by `ChainFamily`
- **Connection chain family inquiry** — PortfolioAggregation must be able to determine which chain families are active for a given connection

### PortfolioAggregation → Blockchain Integrations (Extended)

- **Chain-family-tagged requests** — `AddressRequest` data crossing this boundary must include `chainFamily: ChainFamily`, which determines which integration bounded context handles the request

**Routing rule:** PortfolioAggregation groups tracked addresses by `chainFamily` and dispatches each group to the corresponding integration bounded context.

### Unchanged Contracts

- **App → PortfolioAggregation**: en-fr0z's per-account/per-wallet queries work naturally with multi-chain accounts
- **PortfolioAggregation → Asset Valuator**: Asset valuation is chain-family-agnostic
- **PortfolioAggregation → CEX Integrations**: CEX integrations are independent of wallet connections

---

## 7. Integration Routing

With multi-chain wallets, PortfolioAggregation routes address requests by chain family:

```
PortfolioAggregation receives TrackedAddress[]
    │
    ├── Group by chainFamily
    │
    ├── chainFamily=evm → evm-integration.getBalances()
    ├── chainFamily=solana → sol-integration.getBalances()
    ├── chainFamily=sui → (future) sui-integration.getBalances()
    ├── chainFamily=bitcoin → (future) btc-integration.getBalances()
    ├── chainFamily=cosmos → (future) cosmos-integration.getBalances()
    │
    ├── Combine results, attribute to AccountIds
    │
    ▼
    Portfolio with multi-chain, multi-account attribution
```

### Cross-Chain-Family Deduplication Rules

- **Same wallet, different chain families**: NEVER duplicates. An EVM address and a Solana address from Phantom are distinct holdings on distinct chains.
- **Same chain family, different wallets**: Deduplication applies per en-fr0z rules (same EVM address across two wallet connections → deduplicate in aggregate).
- **Deduplication scope**: Within same chain family only.

---

## 8. Chain Scope Semantics by Chain Family

`chainScope: ChainId[]` from en-fr0z has different semantics per chain family. This is a cross-domain concern because it affects how PortfolioAggregation dispatches requests to integrations:

| Chain Family | Chain Scope Behavior |
|-------------|---------------------|
| **EVM** | One address, many chains. User-configurable list of EVM chains to track. |
| **Solana** | One address, one chain (mainnet). Not user-configurable. |
| **Sui** | One address, one chain (mainnet). Not user-configurable. |
| **Bitcoin** | One address, one chain (mainnet). Not user-configurable. |
| **Cosmos** | One address per chain. User selects which Cosmos chains to enable. |
| **Aptos** | One address, one chain (mainnet). Not user-configurable. |

---

## 9. Migration Path

### Schema Versioning

- en-fr0z Version 2: Multi-wallet multi-account model
- en-o8w Version 3: Multi-chain wallet unification model

### Cross-Domain Migration Impact

- **AccountId format**: Existing `{connectionId}:{address}` → `{connectionId}:evm:{address}`. All domains consuming AccountIds must handle the new format.
- **WalletConnection**: Existing connections gain `supportedChainFamilies: ['evm']`. No data loss.
- **WatchAddress**: Existing watch addresses gain `chainFamily` inferred from address format or `chainScope`.
- **Existing single-chain wallets** (MetaMask, Rabby, etc.) continue working unchanged — multi-chain machinery is not activated for them.

Domain-internal migration logic (how each domain handles the schema upgrade) is owned by Domain Arch.

---

## 10. Interaction with Existing Directives

### en-fr0z (Multi-Wallet Multi-Account Architecture)

This directive is a **superset extension** of en-fr0z. Every type, contract, and pattern from en-fr0z remains valid. This directive adds the `ChainFamily` dimension.

### en-p0ha (WebSocket-First Architecture)

Subscription events gain `chainFamily` context alongside existing `chainId` and `accountIds`. No fundamental architecture change.

### en-tfkn / en-25w5 (RPC Provider Strategy)

RPC infrastructure is per-chain, not per-chain-family. Multi-chain wallets generate more addresses to track, but RPC routing is unchanged.

### hq-onegb / hq-su5pt (CEX Integrations)

No change. CEX integrations are independent of wallet connections.

---

## 11. Security Guarantees (Cross-Domain)

All en-fr0z security guarantees carry forward across all chain families:
- **No private key access** across any chain family
- **No transaction signing** — never request signing methods on any chain
- **Minimal permissions** — only account/address read permissions
- **Session data encryption** — all persisted data encrypted via EncryptionService

WalletConnect v2 session proposals MUST include ONLY read-only methods. No signing methods are permitted.

---

## 12. Open Questions

### Cosmos Chain Selection

Cosmos is an ecosystem of chains, each requiring separate connection calls. Whether to auto-enable all Cosmos chains or let users select is a cross-domain UX decision affecting both WalletIntegration and PortfolioAggregation routing.

**Recommendation**: Let users select Cosmos chains, defaulting to Cosmos Hub.

### Future Chain Family Extensibility

The `ChainFamily` enum is deliberately closed. Adding new families (e.g., `'polkadot'`, `'tron'`, `'near'`) requires Enterprise Arch approval: a new enum value, a new integration bounded context, and correlation registry entries. This is intentional — each chain family is a meaningful architectural boundary.

---

## Cross-Domain Contract Impact Summary

| Contract | Change from en-fr0z | Details |
|----------|---------------------|---------|
| App → Wallet Integration | **Extended** | Discovery returns `DiscoveredWallet` with chain family capabilities; connection options include chain family selection |
| Portfolio Aggregation → Wallet Integration | **Extended** | `TrackedAddress` includes `chainFamily`; chain-family filtering and connection inquiry capabilities added |
| Portfolio Aggregation → Blockchain Integrations | **Extended** | `AddressRequest` includes `chainFamily` for routing; routing dispatches to correct integration BC |
| App → Portfolio Aggregation | **No change** | Per-account/per-wallet queries work naturally with multi-chain accounts |
| Portfolio Aggregation → Asset Valuator | **No change** | Asset valuation is chain-family-agnostic |
| Portfolio Aggregation → CEX Integrations | **No change** | CEX integrations are independent of wallet connections |
| Data Models | **Extended** | New `ChainFamily` type, `Caip2ChainId` type; existing types gain `chainFamily` field |
