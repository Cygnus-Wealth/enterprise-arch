# en-7hfo: EIP-6963 Wallet Discovery Integration

**Status**: RECOMMENDATION
**Date**: 2026-02-25
**Bead**: en-7hfo
**Builds On**: en-o8w (Multi-Chain Wallet Unification), en-fr0z (Multi-Wallet Multi-Account Architecture)
**Scope**: Cross-domain integration contract for EIP-6963 wallet discovery between the Experience domain (CygnusWealthApp) and the Integration domain (WalletIntegrationSystem). Defines how CWA consumes WIS's discovery service, migration from legacy detection, branding data flow, and fallback behavior.

---

## Executive Summary

**CygnusWealthApp uses legacy `window.ethereum` property flags (`isMetaMask`, `isCoinbaseWallet`, etc.) to detect wallets. This is unreliable — when multiple EVM wallets are installed, they fight over `window.ethereum` and the wrong wallet opens. WalletIntegrationSystem already implements EIP-6963 discovery but CWA doesn't consume it.**

en-o8w mandates EIP-6963 for all EVM wallet detection. This directive specifies the cross-domain integration contract that fulfills that mandate: how CWA obtains discovered wallets from WIS, how wallet branding (icons, names, RDNS identifiers) flows to the UI, and how the transition from legacy flag-based detection to event-based discovery is managed.

### Key Decisions

1. **CWA consumes WIS discovery directly** — not through PortfolioAggregation. Discovery is a pre-connection concern; portfolio routing is post-connection. The dependency path is Experience → Integration, which is an existing sanctioned boundary crossing.
2. **EIP-6963 is the primary discovery path for EVM wallets.** Legacy `window.ethereum` detection is retained as a fallback for wallets that haven't adopted EIP-6963.
3. **Wallet branding from EIP-6963 announcements replaces hardcoded wallet metadata in CWA.** The UI renders provider-supplied names and icons rather than maintaining its own wallet icon registry.

---

## 1. The Problem

### Current State

CygnusWealthApp detects wallets by probing `window.ethereum` and checking boolean property flags:

```
window.ethereum.isMetaMask → "This is MetaMask"
window.ethereum.isCoinbaseWallet → "This is Coinbase Wallet"
window.ethereum.isRabby → "This is Rabby"
```

This approach has three cross-domain failures:

1. **Provider collision**: Multiple wallets overwrite `window.ethereum`. MetaMask and Rabby both set `isMetaMask = true`. The last wallet to inject wins, and the user has no control over which wallet opens.
2. **Hardcoded wallet registry**: CWA maintains a static list of known wallet property flags. New wallets (Slush, Frame, etc.) require CWA code changes to detect. WIS already discovers these wallets via EIP-6963 but CWA doesn't ask.
3. **Missing branding**: CWA uses its own hardcoded icons and names for wallets. EIP-6963 announcements include provider-supplied icons, names, and RDNS identifiers that CWA ignores.

### Desired State

CWA asks WIS "what wallets are available?" and receives a list of `DiscoveredWallet` entries (as defined in en-o8w) with full branding metadata. No `window.ethereum` probing. No hardcoded property flags. No static wallet registry in the Experience domain.

---

## 2. Domain Boundary: Discovery Data Flow

### Dependency Direction

```
Experience (CWA) ──depends on──▶ Integration (WIS)
```

CWA depends on WIS for wallet discovery. This is a direct Experience → Integration dependency. This does NOT route through PortfolioAggregation because:

- Discovery occurs before connection, before any portfolio data exists
- PortfolioAggregation's responsibility begins after addresses are tracked (en-o8w Section 4)
- Routing discovery through PortfolioAggregation would violate its bounded context — it owns routing of tracked addresses, not provider enumeration

This dependency direction is consistent with en-fr0z and en-o8w, which both define App → WalletIntegration contract boundaries.

### What Crosses the Boundary

The following data crosses from Integration to Experience:

| Data | Direction | Description |
|------|-----------|-------------|
| Discovered wallet list | WIS → CWA | `DiscoveredWallet[]` as defined in en-o8w, enriched with EIP-6963 branding |
| Discovery status | WIS → CWA | Whether discovery is complete, in progress, or requires fallback |
| Connect request | CWA → WIS | User's intent to connect a specific discovered wallet |
| Connection result | WIS → CWA | Success/failure of connection attempt with resulting `WalletConnection` |

### What Does NOT Cross the Boundary

- **EIP-6963 event details**: CWA never listens for `eip6963:announceProvider` events directly. WIS owns all provider discovery protocols (en-o8w Section 4).
- **Provider objects**: CWA never receives raw EIP-1193 provider references. WIS mediates all provider interactions.
- **Correlation logic**: How WIS matches an EIP-6963 EVM provider with a Wallet Standard Solana provider from the same wallet is WIS-internal (en-o8w Section 4).
- **Fallback detection logic**: Whether WIS falls back to `window.ethereum` probing for a particular wallet is a WIS implementation detail.

---

## 3. Cross-Domain Contract: Wallet Discovery

### Discovery Capabilities Required at the Boundary

Building on en-o8w Section 6 ("App → WalletIntegration"), this directive specifies additional capabilities required for EIP-6963 integration:

**Query capabilities CWA must be able to exercise:**

1. **Enumerate discovered wallets** — Retrieve all wallets detected across all discovery protocols (EIP-6963, Wallet Standard, `window.ethereum` fallback), returned as `DiscoveredWallet[]` with branding metadata
2. **Query wallet discovery source** — For any `DiscoveredWallet`, determine whether it was detected via EIP-6963, Wallet Standard, legacy injection, or WalletConnect v2
3. **Query discovery readiness** — Determine whether initial discovery is complete (EIP-6963 is event-based and asynchronous; wallets announce at different times)

**Event capabilities WIS must provide:**

1. **Wallet discovered** — Notify CWA when a new wallet is detected during the discovery window (wallets may announce after initial page load)
2. **Wallet removed** — Notify CWA when a previously discovered wallet is no longer available (browser extension disabled, removed)
3. **Discovery settled** — Notify CWA when the discovery window has elapsed and no further announcements are expected

**Command capabilities CWA must be able to exercise:**

1. **Request discovery refresh** — Trigger a new round of provider discovery (re-dispatch `eip6963:requestProvider` and re-probe standards)
2. **Connect discovered wallet** — Initiate connection to a specific `DiscoveredWallet` by its `WalletProviderId` (existing en-fr0z/en-o8w connect path, unchanged)

---

## 4. EIP-6963 Branding Data

### Ubiquitous Language Extension

**`WalletProviderInfo`** — Branding metadata from an EIP-6963 announcement. Contains:
- **`name`**: Human-readable wallet name (e.g., "MetaMask", "Rabby Wallet")
- **`icon`**: Data URI (SVG or PNG) of the wallet's icon, as provided by the wallet extension
- **`rdns`**: Reverse domain name system identifier (e.g., `io.metamask`, `io.rabby`)
- **`uuid`**: Unique session identifier for this provider instance

This is a subset of the EIP-6963 `EIP6963ProviderInfo` type. It crosses the Integration → Experience boundary as part of `DiscoveredWallet`.

### Branding Data Flow

```
Wallet Extension
    │
    ▼ (eip6963:announceProvider event)
WIS Discovery Layer (Integration domain)
    │
    ├── Extracts WalletProviderInfo (name, icon, rdns, uuid)
    ├── Correlates with other discovery protocols (en-o8w)
    ├── Produces DiscoveredWallet with branding attached
    │
    ▼ (cross-domain boundary)
CWA (Experience domain)
    │
    ├── Renders wallet name from WalletProviderInfo.name
    ├── Renders wallet icon from WalletProviderInfo.icon
    ├── Uses rdns for stable wallet identification across sessions
    │
    ▼
User sees accurate, provider-supplied wallet branding
```

### Cross-Domain Rule: CWA MUST NOT Maintain a Wallet Icon Registry

CWA currently hardcodes wallet icons and names. With EIP-6963 integration:

- **EIP-6963 detected wallets**: CWA renders the `icon` and `name` from `WalletProviderInfo`. No fallback to hardcoded icons.
- **Legacy-detected wallets** (fallback path): CWA may retain a minimal fallback icon set for wallets detected via `window.ethereum` that lack EIP-6963 branding. This fallback set is expected to shrink over time as wallet adoption of EIP-6963 increases.
- **WalletConnect v2 wallets**: Branding comes from the WalletConnect registry metadata, not EIP-6963. WIS provides this branding through the same `DiscoveredWallet` interface.

---

## 5. Fallback Behavior

### Fallback Policy (Cross-Domain)

en-o8w Section 2 establishes: "`window.ethereum` detection remains as fallback for EVM wallets that haven't adopted EIP-6963."

This directive clarifies the cross-domain implications of that fallback:

1. **WIS determines fallback necessity.** If a wallet is detected via `window.ethereum` but not via EIP-6963, WIS includes it in the discovered wallet list with a discovery source indicating legacy detection. CWA does not independently probe `window.ethereum`.

2. **Fallback wallets have degraded branding.** Legacy-detected wallets lack `WalletProviderInfo` (no icon data URI, no rdns, no uuid). CWA must handle this gracefully — displaying a generic wallet icon or a minimal hardcoded fallback for known legacy wallets.

3. **Fallback wallets have degraded disambiguation.** When multiple wallets fight over `window.ethereum`, WIS cannot reliably identify which wallet is which via legacy detection. EIP-6963 wallets are individually identified by uuid/rdns. This disambiguation gap is inherent to legacy detection and cannot be solved at the architecture level.

4. **Fallback is transitional.** As of early 2026, all major EVM wallets support EIP-6963 (MetaMask, Rabby, Coinbase Wallet, Phantom, Brave, Trust Wallet). The legacy fallback exists for edge cases and long-tail wallets. Domains should design for EIP-6963 as the primary path.

### Supported Wallet Coverage

| Wallet | EIP-6963 | Wallet Standard | Legacy Fallback |
|--------|----------|-----------------|-----------------|
| MetaMask | Yes | — | `isMetaMask` |
| Rabby | Yes | — | `isRabby` |
| Coinbase Wallet | Yes | — | `isCoinbaseWallet` |
| Brave Wallet | Yes | — | `isBraveWallet` |
| Phantom (EVM) | Yes | Yes (Solana) | `isPhantom` |
| Trust Wallet | Yes | Yes (Multi-chain) | `isTrust` |
| Slush | Yes | Yes (Sui) | — |
| Suiet | — | Yes (Sui) | — |

All currently supported wallets have EIP-6963 or Wallet Standard support. Pure legacy-only detection is not required for any wallet in the current supported set.

---

## 6. Migration Path

### Phase 1: Dual-Path Discovery (No Breaking Changes)

WIS already implements EIP-6963. The migration is on the CWA side:

1. CWA adds consumption of WIS's discovery capabilities alongside its existing legacy detection
2. When WIS returns an EIP-6963-discovered wallet, CWA uses it (with provider-supplied branding)
3. When WIS returns a legacy-detected wallet, CWA falls back to existing behavior
4. No changes to persisted wallet connections — `WalletConnection` entities are unaffected

**Cross-domain impact**: None. WIS's public discovery capabilities are additive. CWA consumes new data but existing flows are unchanged.

### Phase 2: Primary EIP-6963 (Deprecate Legacy UI)

1. CWA uses WIS discovery as its sole source of wallet enumeration
2. CWA removes `window.ethereum` property flag probing from its own code
3. CWA wallet selection UI is driven entirely by `DiscoveredWallet[]` from WIS
4. Legacy fallback still exists in WIS for wallets that need it, but CWA doesn't distinguish — it renders whatever WIS provides

**Cross-domain impact**: CWA's dependency on WIS discovery changes from optional/parallel to required. WIS must ensure discovery capabilities are reliable and performant.

### Phase 3: Remove Legacy Wallet Registry

1. CWA removes hardcoded wallet icon/name registry
2. All wallet branding comes from `WalletProviderInfo` (EIP-6963/Wallet Standard) or WalletConnect registry metadata
3. Minimal generic fallback icon retained for truly unknown wallets

**Cross-domain impact**: CWA no longer needs to be updated when new wallets are added. WIS's discovery automatically surfaces new wallets with their own branding.

### Backwards Compatibility

- **Existing saved wallet connections**: Unaffected. `WalletConnection` entities persist with their existing `walletProviderId`. Migration adds `WalletProviderInfo` to connected wallets as they are re-discovered via EIP-6963, but the connection itself is unchanged.
- **Reconnection on page load**: Existing connections are restored by matching `walletProviderId`. EIP-6963 rdns provides a more reliable identifier for matching than legacy property flags.

---

## 7. Impact on CWA Wallet State Management

### Cross-Domain Concern

CWA maintains wallet state (connected wallets, selected accounts, connection status). This directive affects what data enters that state, not how the state is structured internally (that is Experience Domain Arch's concern).

**What changes at the boundary:**

1. **Wallet list source**: CWA's available-wallets list must come from WIS discovery, not from a hardcoded registry within CWA
2. **Branding data**: Each wallet entry includes `WalletProviderInfo` branding from the discovery source
3. **Discovery source metadata**: Each wallet entry includes how it was discovered (EIP-6963, Wallet Standard, legacy, WalletConnect), enabling CWA to make UI decisions (e.g., showing a "legacy detected" indicator)

**What does NOT change at the boundary:**

- Connection lifecycle (connect/disconnect commands per en-fr0z)
- Account discovery within a connected wallet (per en-fr0z)
- Multi-chain connection per en-o8w
- Portfolio data flow (PortfolioAggregation → CWA, unchanged)

### Wallet Selection UI Pattern (Cross-Domain Guidance)

Enterprise Arch provides this guidance on the wallet selection pattern because it spans the Experience → Integration boundary:

- **Wallets are presented as discovered by WIS**, not hardcoded. If a wallet extension is not installed, it does not appear (EIP-6963 only announces installed wallets).
- **Multi-chain wallets appear once**, with chain family badges per en-o8w. Phantom appears as one entry showing EVM + Solana capabilities, not as two separate entries.
- **The UI should not expose discovery internals** (EIP-6963 vs Wallet Standard vs legacy). Users don't care how a wallet was detected. However, CWA may use discovery source internally for fallback icon decisions.

Internal wallet selection UI design (layout, animations, filtering, sorting) is the Experience domain's concern.

---

## 8. Changes to WIS Public API Surface

### New Capabilities WIS Must Expose

en-o8w Section 6 defines general "discovered wallet enumeration" and "discovery completion" capabilities. This directive adds specificity:

1. **`WalletProviderInfo` in `DiscoveredWallet`**: Each `DiscoveredWallet` crossing the boundary must include the EIP-6963 branding fields (name, icon, rdns, uuid) when available, or null/absent when the wallet was detected via a protocol that doesn't provide them.

2. **Discovery source indicator**: Each `DiscoveredWallet` must indicate its discovery protocol source, enabling consuming domains to distinguish EIP-6963-detected wallets from legacy-detected ones.

3. **Late announcement handling**: EIP-6963 is event-driven. Wallets may announce after initial discovery. WIS must surface late-arriving wallets to CWA through the existing "wallet discovered" event capability (en-o8w Section 6).

### Unchanged WIS Capabilities

- Connection management (connect/disconnect per en-fr0z)
- Multi-chain connection (per en-o8w)
- Provider correlation (WIS-internal per en-o8w Section 4)
- WalletConnect v2 session management (per en-o8w)

---

## 9. Interaction with Existing Directives

### en-o8w (Multi-Chain Wallet Unification)

This directive is a **focused elaboration** of en-o8w Section 2 (EIP-6963 mandate) and Section 6 (App → WalletIntegration contract). It adds specificity to the discovery integration contract without contradicting any en-o8w decisions.

### en-fr0z (Multi-Wallet Multi-Account Architecture)

en-fr0z defined the connection and account discovery contracts. This directive addresses the pre-connection phase: how CWA learns which wallets are available before the user chooses to connect one. The connection flow itself (en-fr0z) is unchanged.

### en-p0ha (WebSocket-First Architecture)

Discovery events (wallet announced, wallet removed, discovery settled) are part of the WIS → CWA event contract. Whether these use WebSocket channels or direct callbacks is a Domain Arch decision. Enterprise Arch requires only that the event capabilities exist.

---

## 10. Security Guarantees

All en-fr0z and en-o8w security guarantees carry forward:

- **No private key access** — discovery only enumerates available wallets, it does not access keys
- **No transaction signing** — discovery never invokes signing methods
- **Icon data sanitization** — EIP-6963 icons are data URIs supplied by wallet extensions. CWA must treat them as untrusted content (render in sandboxed contexts, validate data URI format). This is an Experience domain implementation concern but is flagged here as a cross-domain security requirement because the data originates in the Integration domain.
- **RDNS as identifier, not authority** — The `rdns` field is self-reported by the wallet extension and is not cryptographically verified. It is useful for matching and display but must not be used as a security assertion.

---

## Cross-Domain Contract Impact Summary

| Contract | Change | Details |
|----------|--------|---------|
| App → WalletIntegration (Discovery) | **New** | CWA consumes `DiscoveredWallet[]` with `WalletProviderInfo` branding from WIS discovery |
| App → WalletIntegration (Connection) | **No change** | Connection lifecycle per en-fr0z/en-o8w unchanged |
| App → PortfolioAggregation | **No change** | Portfolio queries unchanged; discovery is pre-connection |
| WalletIntegration → PortfolioAggregation | **No change** | TrackedAddress flow per en-o8w unchanged |
| Data Models | **Extended** | `DiscoveredWallet` type gains `WalletProviderInfo` fields and discovery source indicator |
| CWA Internal State | **Affected** | Wallet list source changes from hardcoded registry to WIS discovery results (internal design is Domain Arch concern) |
