# en-o8w: Multi-Chain Wallet Unification

**Status**: RECOMMENDATION
**Date**: 2026-02-25
**Bead**: en-o8w
**Builds On**: en-fr0z (Multi-Wallet Multi-Account Architecture)
**Scope**: Extend the wallet identity model from en-fr0z to support wallets that span multiple chain families (EVM, Solana, Sui, Bitcoin, Cosmos, Aptos). Defines provider discovery standards (EIP-6963, Wallet Standard), WalletConnect v2 multi-namespace sessions, chain capability detection, and cross-domain impact.

---

## Executive Summary

**A single wallet like Trust Wallet or Phantom exposes addresses on multiple chain families. The system must treat these as one unified wallet entity, not as unrelated connections.**

en-fr0z established the multi-wallet multi-account identity model: `WalletConnection → ConnectedAccount[]`. That directive is EVM-focused — it assumes EIP-1193 providers, `eth_requestAccounts`, and `accountsChanged` events. When a user connects Trust Wallet, the system currently sees the EVM side only. The Solana, Cosmos, and Aptos sides either appear as separate unrelated connections or are invisible.

This directive extends en-fr0z to unify multi-chain wallets under a single `WalletConnection`, correlate chain-specific sub-providers from the same wallet, adopt modern discovery standards (EIP-6963 for EVM, Wallet Standard for Solana/Sui/Aptos), and support WalletConnect v2 multi-namespace sessions.

### Key Constraint: Read-Only, Client-Side Only

All constraints from en-fr0z carry forward. We never access private keys, sign transactions, or make server calls. Multi-chain unification is purely about provider discovery, address correlation, and identity modeling.

---

## 1. The Multi-Chain Problem

### How Multi-Chain Wallets Inject Providers

Each wallet injects chain-specific sub-providers. There is no universal cross-chain provider API. Every wallet does this differently:

| Wallet | EVM Provider | Solana Provider | Bitcoin Provider | Other Providers |
|--------|-------------|-----------------|------------------|-----------------|
| **Trust Wallet** | `window.trustwallet.ethereum` (+ EIP-6963) | `window.trustwallet.solana` (+ Wallet Standard) | — | Cosmos: `window.trustwallet.cosmos`, Aptos: `window.trustwallet.aptos` |
| **Phantom** | `window.phantom.ethereum` (+ EIP-6963) | `window.phantom.solana` (+ Wallet Standard) | `window.phantom.bitcoin` | — |
| **OKX Wallet** | `window.okxwallet` (+ EIP-6963) | `window.okxwallet.solana` (+ Wallet Standard) | `window.okxwallet.bitcoin` | — |
| **Backpack** | EIP-6963 | Wallet Standard | — | Sui: Wallet Standard |
| **Crypto.com DeFi** | `window.deficonnectProvider` (EIP-6963) | — | — | Cosmos: `window.deficonnectProvider` (chainType toggle) |
| **WalletConnect v2** | eip155 namespace | solana namespace | bip122 namespace | cosmos, polkadot namespaces |

### What Goes Wrong Without Unification

1. User connects Phantom. System creates an EVM connection (MetaMask-like flow via EIP-1193). Separately, user connects Phantom for Solana. System creates a second, unrelated connection. The user now sees "Phantom" twice with no way to know they are the same wallet.

2. User connects Trust Wallet. Only the EVM side is detected (via `window.ethereum` or EIP-6963). The Solana, Cosmos, and Aptos addresses from the same wallet are invisible — the user must manually add them as watch addresses.

3. User connects via WalletConnect v2 and approves EVM + Solana chains in the same session. The system creates one EVM connection and one Solana connection with no link between them.

### The Goal

A single Trust Wallet connection should surface:
- EVM addresses (Ethereum, Polygon, Arbitrum, etc.)
- Solana addresses
- Cosmos addresses
- Aptos addresses

All under **one** `WalletConnection` entity, labeled "Trust Wallet", with chain-family-aware accounts.

---

## 2. Provider Discovery Architecture

### The Discovery Problem

Before en-fr0z, CygnusWealth detected wallets via `window.ethereum` — a global that wallets race to claim. This is unreliable when multiple wallets are installed. Modern standards solve this:

- **EIP-6963** (Multi Injected Provider Discovery): Standardized EVM provider announcement via `eip6963:announceProvider` events. Each wallet announces itself with a UUID, name, icon, and RDNS identifier. No global race condition.
- **Wallet Standard** (`@wallet-standard/base`): Standardized wallet registration for Solana, Sui, and Aptos. Wallets register via `window.addEventListener('wallet-standard:register-wallet', ...)` and expose chain-specific features.
- **Bitcoin**: No finalized standard. Detection is wallet-specific (`window.phantom.bitcoin`, `window.okxwallet.bitcoin`). The Sats Connect / WebBTC proposals are not yet widely adopted.
- **WalletConnect v2**: Session-based — chains are negotiated during pairing, not discovered from injected providers.

### Unified Discovery Layer

The WalletIntegrationSystem introduces a `WalletDiscoveryService` that bridges all discovery protocols and correlates providers from the same wallet:

**WalletDiscoveryService Responsibilities:**

1. Listen for EIP-6963 announcements (EVM providers)
2. Listen for Wallet Standard registrations (Solana/Sui/Aptos providers)
3. Probe known global injection points for Bitcoin providers
4. Correlate providers from the same wallet into a unified `DiscoveredWallet`
5. Expose a single `getAvailableWallets()` query that returns correlated, multi-chain wallet entries

### Provider Correlation

The critical challenge: EIP-6963 and Wallet Standard are separate protocols. Phantom registers as an EIP-6963 EVM provider AND as a Wallet Standard Solana provider. These are two independent announcements with no built-in cross-reference.

**Correlation strategy — RDNS matching:**

EIP-6963 providers include an `rdns` field (reverse domain name, e.g., `app.phantom`, `com.trustwallet.app`). Wallet Standard wallets include a `name` field. We maintain a correlation registry that maps known RDNS values and wallet names to a canonical `WalletProviderId`:

```
CorrelationRegistry {
  // Maps EIP-6963 RDNS → canonical provider ID
  eip6963RdnsMap: {
    'app.phantom': 'phantom',
    'com.trustwallet.app': 'trust-wallet',
    'com.okex.wallet': 'okx-wallet',
    'app.backpack': 'backpack',
    'com.crypto.wallet': 'crypto-com-defi',
  }

  // Maps Wallet Standard name → canonical provider ID
  walletStandardNameMap: {
    'Phantom': 'phantom',
    'Trust': 'trust-wallet',
    'OKX Wallet': 'okx-wallet',
    'Backpack': 'backpack',
  }

  // Maps known global injection points → canonical provider ID + chain family
  globalInjectionMap: {
    'window.phantom.bitcoin': { providerId: 'phantom', chainFamily: 'bitcoin' },
    'window.okxwallet.bitcoin': { providerId: 'okx-wallet', chainFamily: 'bitcoin' },
    'window.trustwallet.cosmos': { providerId: 'trust-wallet', chainFamily: 'cosmos' },
    'window.trustwallet.aptos': { providerId: 'trust-wallet', chainFamily: 'aptos' },
  }
}
```

**Fallback for unknown wallets:**

If an EIP-6963 provider's RDNS is not in the registry, it appears as an EVM-only wallet (same as today). If a Wallet Standard wallet's name doesn't match any known RDNS, it appears as a standalone Solana/Sui wallet. Users can manually link connections later (see Section 6 — UX).

**Registry maintenance:**

The correlation registry is a static data structure shipped with the application. It is updated when new multi-chain wallets are added to the supported list. The registry is NOT fetched from a remote server (client-side sovereignty).

### DiscoveredWallet Type

The discovery layer produces `DiscoveredWallet` objects:

```
DiscoveredWallet {
  walletId: WalletProviderId              // canonical ID (e.g., 'phantom')
  displayName: string                     // human-readable (e.g., "Phantom")
  icon: string                            // provider icon URL or data URI
  chainFamilies: ChainFamilyCapability[]  // which chains this wallet supports
  isMultiChain: boolean                   // true if > 1 chain family
}

ChainFamilyCapability {
  chainFamily: ChainFamily                // 'evm', 'solana', 'sui', 'bitcoin', 'cosmos', 'aptos'
  discoverySource: DiscoverySource        // how this capability was found
  providerRef: any                        // reference to the chain-specific provider object
  supportedChains: ChainId[]              // specific chains within the family
  features: string[]                      // e.g., ['signMessage', 'signTransaction'] — we only use read-only features
}

ChainFamily = 'evm' | 'solana' | 'sui' | 'bitcoin' | 'cosmos' | 'aptos'

DiscoverySource = 'eip6963' | 'wallet-standard' | 'global-injection' | 'walletconnect'
```

### Discovery Flow

1. **EIP-6963 phase**: Dispatch `eip6963:requestProvider`. Collect all `eip6963:announceProvider` events. Each announcement becomes a pending EVM provider keyed by its RDNS.

2. **Wallet Standard phase**: Call `getWallets()` from `@wallet-standard/app`. Each registered wallet is inspected for supported features (`standard:connect`, `solana:signTransaction`, `sui:signTransaction`, etc.). The wallet name is checked against the correlation registry.

3. **Global injection probe**: For each entry in `globalInjectionMap`, check if the global object exists. If found, record the capability for the corresponding provider.

4. **Correlation phase**: Group all discovered providers by canonical `WalletProviderId`. Each group becomes a `DiscoveredWallet` with its combined `chainFamilies`.

5. **Emit**: The discovery service emits `onWalletsDiscovered` with the full list of `DiscoveredWallet` entries.

### EIP-6963 Adoption Details

EIP-6963 replaces the `window.ethereum` global with an event-based announcement protocol:

**Listening:**
- On initialization, add listener for `eip6963:announceProvider` events
- Dispatch `eip6963:requestProvider` to trigger announcements from already-loaded wallets
- Each event contains `{ info: { uuid, name, icon, rdns }, provider: EIP1193Provider }`

**Advantages over window.ethereum:**
- No race condition — all wallets announce independently
- Each wallet is uniquely identified by UUID and RDNS
- Provider icon and name are standardized
- Multiple EVM wallets coexist without conflict

**Migration from window.ethereum:**
- `window.ethereum` detection remains as a fallback for wallets that haven't adopted EIP-6963
- If a wallet is detected via both EIP-6963 and `window.ethereum`, the EIP-6963 provider takes precedence
- The fallback is logged so we can track EIP-6963 adoption gaps

### Wallet Standard Adoption Details

The Wallet Standard provides a registration-based discovery protocol used by Solana, Sui, and Aptos wallets:

**Listening:**
- Call `getWallets()` from `@wallet-standard/app` to get currently registered wallets
- Subscribe to `register` and `unregister` events for dynamic changes

**Feature detection:**
- Each wallet exposes a `features` map (e.g., `standard:connect`, `solana:signAndSendTransaction`)
- We only require `standard:connect` — we never use signing features
- Chain support is inferred from feature namespaces: `solana:*` means Solana support, `sui:*` means Sui support

**Account model:**
- The Wallet Standard `connect` method returns `Account[]` objects with `address`, `publicKey`, and `chains` (CAIP-2 format: `solana:mainnet`, `sui:mainnet`)
- This maps directly to our `ConnectedAccount` with `chainFamily` and `chainScope`

---

## 3. Extended Identity Model

### Changes to en-fr0z Types

en-fr0z defined `WalletConnection`, `ConnectedAccount`, `WatchAddress`, and supporting types for EVM-only multi-wallet support. This directive extends those types for multi-chain:

**`ChainFamily`** — new enum for chain family classification:
- Values: `'evm'`, `'solana'`, `'sui'`, `'bitcoin'`, `'cosmos'`, `'aptos'`
- This is distinct from `ChainId` (which identifies a specific chain within a family, e.g., `ethereum`, `polygon`, `solana-mainnet`)

**`WalletProviderId`** — extended with new multi-chain wallets:
- Add: `'okx-wallet'`, `'backpack'`
- Existing values from en-fr0z unchanged

**`WalletConnection`** — extended with chain family awareness:
- New field: `supportedChainFamilies: ChainFamily[]` — all chain families this wallet connection can provide addresses for
- New field: `chainProviders: Map<ChainFamily, ChainProviderRef>` — references to the chain-specific provider objects (EIP-1193 provider for EVM, Wallet Standard wallet for Solana, etc.)
- `supportedChains: ChainId[]` now computed: the union of all chains across all chain families
- All existing en-fr0z fields unchanged

**`ChainProviderRef`** — reference to a chain-family-specific provider:
- `chainFamily: ChainFamily`
- `discoverySource: DiscoverySource`
- `providerRef: any` — opaque reference to the provider object (EIP-1193 provider, Wallet Standard wallet, or Bitcoin provider)
- `isConnected: boolean` — whether this chain family's provider is currently connected
- `supportedChains: ChainId[]` — specific chains available via this provider

**`ConnectedAccount`** — extended with chain family:
- New field: `chainFamily: ChainFamily` — which chain family this address belongs to
- `chainScope: ChainId[]` semantics unchanged: specific chains within the family to track
- `AccountId` format updated (see below)
- All existing en-fr0z fields unchanged

**`AccountId`** — format extended for chain-family namespacing:
- Old format (en-fr0z): `{walletConnectionId}:{checksummedAddress}`
- New format: `{walletConnectionId}:{chainFamily}:{address}`
  - Example EVM: `phantom:a1b2c3d4:evm:0xAbC...123` (where `phantom:a1b2c3d4` is the full WalletConnectionId per en-fr0z format `{providerId}:{randomId}`)
  - Example Solana: `phantom:a1b2c3d4:solana:7nYB...xyz`
  - Example Bitcoin: `phantom:a1b2c3d4:bitcoin:bc1q...abc`
  - Watch addresses: `watch:{chainFamily}:{address}`
- The `chainFamily` segment disambiguates addresses that could theoretically collide across chain families (different address formats make this unlikely, but the format is explicit)

**`WatchAddress`** — extended with chain family:
- New field: `chainFamily: ChainFamily` — required when adding a watch address
- All existing en-fr0z fields unchanged

### The Extended Hierarchy

```
User
 └── WalletConnection[]  (one per connected wallet — now potentially multi-chain)
       ├── provider: WalletProviderId           (phantom, trust-wallet, okx-wallet, etc.)
       ├── connectionId: WalletConnectionId
       ├── connectionLabel: string
       ├── supportedChainFamilies: ChainFamily[]  ← NEW (e.g., ['evm', 'solana', 'bitcoin'])
       ├── chainProviders: Map<ChainFamily, ChainProviderRef>  ← NEW
       └── accounts: ConnectedAccount[]
             ├── address: string
             ├── chainFamily: ChainFamily        ← NEW (evm, solana, bitcoin, etc.)
             ├── accountLabel: string
             ├── chainScope: ChainId[]           (chains within the family)
             └── source: 'provider' | 'manual'

 └── WatchAddress[]
       ├── address: string
       ├── chainFamily: ChainFamily              ← NEW
       ├── addressLabel: string
       └── chainScope: ChainId[]
```

### Why One WalletConnection Per Wallet (Not Per Chain Family)

**Alternative considered:** One WalletConnection per chain family per wallet. Phantom would create three connections: one for EVM, one for Solana, one for Bitcoin.

**Rejected because:**
- Users think of Phantom as ONE wallet. Three separate connections confuse the mental model.
- en-fr0z's identity model is per-session, not per-chain. A WalletConnect v2 session that spans EVM + Solana is naturally one session.
- Account grouping (en-fr0z's AccountGroup) becomes awkward — users would need to manually group the three Phantom connections.
- Portfolio attribution ("show me everything in Phantom") requires a separate correlation layer that duplicates what the WalletConnection already provides.

**One connection, multiple chain families** keeps the model from en-fr0z intact and extends it minimally.

---

## 4. Multi-Chain Connection Flow

### Connecting a Multi-Chain Wallet

When the user selects a `DiscoveredWallet` with `isMultiChain: true`:

1. **Chain Family Selection**: The connection modal shows which chain families the wallet supports. The user selects which to connect (default: all supported). This selection determines which chain-specific providers to activate.

2. **Per-Chain-Family Connection**: For each selected chain family, the system calls the appropriate connect method:
   - **EVM** (EIP-6963 provider): `provider.request({ method: 'eth_requestAccounts' })`
   - **Solana** (Wallet Standard): `wallet.features['standard:connect'].connect()`
   - **Sui** (Wallet Standard): `wallet.features['standard:connect'].connect()`
   - **Bitcoin** (global injection): Provider-specific connect method (e.g., Phantom Bitcoin uses `provider.requestAccounts()`)
   - **Cosmos** (global injection): Provider-specific connect (e.g., `provider.enable(chainId)`)
   - **Aptos** (Wallet Standard or global): `wallet.features['standard:connect'].connect()` or `provider.connect()`

3. **Account Collection**: Each chain family's connect method returns addresses specific to that family. These become `ConnectedAccount` entries with the appropriate `chainFamily`.

4. **WalletConnection Assembly**: A single `WalletConnection` is created with all discovered accounts across all connected chain families. The `supportedChainFamilies` field records which families were connected. The `chainProviders` map stores references to each chain-specific provider.

5. **Event Emission**: `onWalletConnected` fires with the full `WalletConnection`. `onAccountDiscovered` fires for each account.

### Per-Chain-Family Account Events

Different chain families use different event models for account changes:

**EVM (EIP-1193):**
- `accountsChanged` event — fires when user switches accounts in wallet UI
- Accumulation model from en-fr0z applies: new accounts are added, not replaced
- `chainChanged` event — fires when user switches EVM chain in wallet UI

**Solana (Wallet Standard):**
- `change` event on the wallet with `accounts` property
- Solana wallets typically expose one account at a time
- Accumulation model applies

**Sui (Wallet Standard):**
- `change` event with `accounts` property
- Same behavior as Solana

**Bitcoin:**
- No standardized event model
- Polling fallback: re-request accounts periodically (every 30 seconds when wallet panel is active)
- Phantom Bitcoin emits events via its provider; OKX does not — wallet-specific adapters handle the difference

**Cosmos:**
- `accountsChanged` event (Keplr/Leap standard)
- `keystorechange` event on `window` for key changes

**WalletConnect v2:**
- `session_update` event — fires when the remote wallet updates approved chains or accounts
- `session_delete` event — fires when the remote wallet disconnects

### Chain-Specific Adapter Pattern

Each chain family requires an adapter that normalizes the connection interface:

```
ChainFamilyAdapter {
  chainFamily: ChainFamily
  connect(providerRef: any): ConnectedAccount[]
  disconnect(providerRef: any): void
  getAccounts(providerRef: any): ConnectedAccount[]
  subscribeToAccountChanges(providerRef: any, callback: (accounts: ConnectedAccount[]) => void): Unsubscribe
  subscribeToChainChanges(providerRef: any, callback: (chainId: ChainId) => void): Unsubscribe
  validateAddress(address: string): boolean
  formatAddress(address: string): string    // checksumming for EVM, base58 validation for Solana, etc.
}
```

Implementations:

- `EvmChainFamilyAdapter` — uses EIP-1193 / EIP-6963 provider
- `SolanaChainFamilyAdapter` — uses Wallet Standard wallet object
- `SuiChainFamilyAdapter` — uses Wallet Standard wallet object
- `BitcoinChainFamilyAdapter` — uses wallet-specific global injection providers
- `CosmosChainFamilyAdapter` — uses wallet-specific global injection providers (Keplr interface)
- `AptosChainFamilyAdapter` — uses Wallet Standard wallet object

### Chain Scope Semantics for Non-EVM Families

en-fr0z's `chainScope: ChainId[]` on `ConnectedAccount` was designed for EVM, where one address is valid across many chains (Ethereum, Polygon, Arbitrum, etc.). For non-EVM chain families, chain scope semantics differ:

- **EVM**: One address, many chains. Chain scope is a user-configurable list of which EVM chains to track. Default: all supported.
- **Solana**: One address, one chain (mainnet). Chain scope defaults to `['solana-mainnet']`. Devnet/testnet may be added for developer users but is not exposed by default.
- **Sui**: One address, one chain (mainnet). Same as Solana.
- **Bitcoin**: One address, one chain (mainnet). Chain scope defaults to `['bitcoin-mainnet']`. No user-facing chain scope picker — Bitcoin does not have meaningful chain scope like EVM.
- **Cosmos**: One address per chain. Chain scope is meaningful — the user selects which Cosmos chains to enable (Cosmos Hub, Osmosis, Celestia, etc.). Each enabled chain produces a separate address via the wallet's `getKey(chainId)` method.
- **Aptos**: One address, one chain (mainnet). Same as Solana/Sui.

**UI implication**: The chain scope picker (from en-fr0z) only appears for EVM and Cosmos accounts. For Solana, Sui, Bitcoin, and Aptos, chain scope is pre-determined and not user-editable.

### Partial Connection Failures

When connecting a multi-chain wallet, individual chain families may fail (user denies permission for one chain, provider crashes, etc.):

- **Partial success is allowed.** If EVM connects but Solana fails, the `WalletConnection` is created with only EVM accounts. `supportedChainFamilies` still lists all families the wallet supports, but `chainProviders` only contains the successfully connected family.
- **Retry per chain family.** The UI offers a "Connect {ChainFamily}" button for each unconnected chain family within an existing wallet connection.
- **No cascading disconnection.** Disconnecting one chain family does NOT disconnect others within the same wallet connection.

### Error Handling in Multi-Chain Connections

**Discovery errors**: If `getWallets()` throws or EIP-6963 discovery times out (500ms), the discovery service falls back to known global injection points only. An error is logged but not surfaced to the user — the wallet list may simply be incomplete.

**Connection errors per chain family**: Each chain family's connect call is independent. Failures produce a `ChainFamilyConnectionError` with `{ chainFamily, errorCode, message }`. Error codes include:
- `USER_REJECTED` — user denied permission in wallet UI
- `PROVIDER_UNAVAILABLE` — provider was discovered but is no longer available (extension disabled, wallet locked)
- `TIMEOUT` — connect call did not respond within 10 seconds
- `UNKNOWN` — unexpected error

**Error surfacing**: The connection modal shows per-chain-family status: green check for connected, red X for failed with error message, spinner for pending. The user can retry individual chain families without restarting the entire connection flow.

**Stale discovery**: If a wallet discovered at time T is unavailable at connection time T+N, the system retries discovery once (re-dispatch EIP-6963 / re-call `getWallets()`). If still unavailable, surface `PROVIDER_UNAVAILABLE` to the user.

---

## 5. WalletConnect v2 Multi-Namespace Sessions

### CAIP-2 Chain Identification

WalletConnect v2 uses CAIP-2 chain identifiers instead of numeric chain IDs:

| Chain | CAIP-2 Identifier |
|-------|-------------------|
| Ethereum Mainnet | `eip155:1` |
| Polygon | `eip155:137` |
| Solana Mainnet | `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp` |
| Bitcoin Mainnet | `bip122:000000000019d6689c085ae165831e93` |
| Cosmos Hub | `cosmos:cosmoshub-4` |

### CAIP-2 to ChainId Mapping

The system maintains a static mapping between CAIP-2 identifiers and our internal `ChainId` type:

```
Caip2Registry {
  caip2ToChainId: {
    'eip155:1': 'ethereum',
    'eip155:137': 'polygon',
    'eip155:42161': 'arbitrum',
    'eip155:10': 'optimism',
    'eip155:56': 'bnb-smart-chain',
    'eip155:43114': 'avalanche',
    'eip155:8453': 'base',
    'solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp': 'solana-mainnet',
    'bip122:000000000019d6689c085ae165831e93': 'bitcoin-mainnet',
    'cosmos:cosmoshub-4': 'cosmos-hub',
  }

  caip2NamespaceToChainFamily: {
    'eip155': 'evm',
    'solana': 'solana',
    'bip122': 'bitcoin',
    'cosmos': 'cosmos',
    'polkadot': 'polkadot',
  }
}
```

### WalletConnect v2 Session as WalletConnection

A WalletConnect v2 session maps naturally to the extended identity model:

1. **Pairing**: User scans QR code or uses deep link. The required/optional namespaces in the session proposal define which chains we request.

2. **Session Approval**: The remote wallet returns a session with approved namespaces. Each namespace contains `accounts` in CAIP-10 format (e.g., `eip155:1:0xAbC...123`, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp:7nYB...xyz`).

3. **Mapping to WalletConnection**:
   - `providerId: 'walletconnect'`
   - `supportedChainFamilies`: derived from approved namespace keys
   - `chainProviders`: one `ChainProviderRef` per namespace, each referencing the WalletConnect session with a namespace filter
   - `accounts`: parsed from CAIP-10 account strings — each becomes a `ConnectedAccount` with `chainFamily` derived from the namespace

4. **Session proposal configuration**:

```
SessionProposal {
  requiredNamespaces: {
    eip155: {
      chains: ['eip155:1', 'eip155:137'],
      methods: ['eth_getBalance'],     // read-only methods only
      events: ['accountsChanged', 'chainChanged']
    }
  }
  optionalNamespaces: {
    solana: {
      chains: ['solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp'],
      methods: ['getBalance'],
      events: []
    }
    bip122: {
      chains: ['bip122:000000000019d6689c085ae165831e93'],
      methods: ['getBalance'],
      events: []
    }
  }
}
```

- EVM is `requiredNamespaces` because it's the most commonly supported chain family.
- Other chain families are `optionalNamespaces` — the remote wallet may or may not support them.
- Only read-only methods are requested. No signing methods are included.

5. **Session events**:
   - `session_update`: The remote wallet changes approved chains or accounts. Update the `WalletConnection` accordingly — add new chain families or accounts, remove revoked ones.
   - `session_delete`: The remote wallet disconnects. Mark the entire `WalletConnection` as `sessionStatus: 'disconnected'`. Accounts transition to watch-like behavior (still tracked for read-only monitoring).

### WalletConnect v2 Peer Metadata

WalletConnect v2 sessions include peer metadata (the remote wallet's name and icon). This is used for:
- `WalletConnection.connectionLabel` default value
- `WalletConnection.providerIcon` value
- `providerId` remains `'walletconnect'` regardless of peer metadata. Peer metadata names are unverified user-controlled strings from the remote wallet and MUST NOT be used for provider identity correlation. The peer's claimed name (e.g., "Trust Wallet") is stored as display metadata only — it does not grant the same trust level as a locally-detected EIP-6963 or Wallet Standard provider. The UI may show "WalletConnect (via Trust Wallet)" to convey the peer's claim without conflating it with a direct local connection.

---

## 6. Chain Capability Detection

### DiscoveredWallet Capabilities

When the discovery layer produces a `DiscoveredWallet`, each `ChainFamilyCapability` includes:
- **`chainFamily`**: The chain family enum value
- **`supportedChains`**: Specific chains within the family. For EVM, this is determined after connection (the wallet may support any EVM chain). For Solana/Sui, typically only mainnet. For Bitcoin, mainnet only.
- **`features`**: Provider features. We only care about read-only features, but exposing the full list allows future extension.

### Pre-Connection vs Post-Connection Capabilities

**Pre-connection** (from discovery only):
- Which chain families the wallet supports (from correlation registry + detected providers)
- Provider metadata (name, icon, RDNS)
- Whether the wallet is multi-chain

**Post-connection** (after user connects):
- Specific accounts/addresses per chain family
- Specific chain support within each family (EVM wallets may support a subset of EVM chains)
- Session health and staleness

The UI shows pre-connection capabilities in the wallet selection modal (e.g., "Phantom — EVM, Solana, Bitcoin") and post-connection details in the wallet management panel.

### Capability Display Format

For the UI, wallet capabilities are displayed as:
- **Wallet name** with chain family icons/badges
- Example: "Trust Wallet" with EVM, Solana, Cosmos, Aptos badges
- Example: "MetaMask" with EVM badge only
- Example: "Suiet" with Sui badge only

Single-chain wallets display identically to today — no change to existing UX for MetaMask, Rabby, Suiet, etc.

---

## 7. Data Model Changes (Contract Domain — data-models)

### New Types

**`ChainFamily`** — enum classifying blockchain families:
- Values: `'evm'`, `'solana'`, `'sui'`, `'bitcoin'`, `'cosmos'`, `'aptos'`
- Used throughout the system for routing, discovery, and display

**`ChainFamilyMetadata`** — static metadata about each chain family:
- `chainFamily: ChainFamily`
- `displayName: string` — human-readable (e.g., "EVM", "Solana", "Bitcoin")
- `addressFormat: string` — description of address format (e.g., "0x-prefixed hex, 20 bytes", "base58, 32-44 chars")
- `addressValidator: (address: string) => boolean` — format validation function
- `icon: string` — chain family icon identifier

**`Caip2ChainId`** — CAIP-2 chain identifier string type:
- Format: `{namespace}:{reference}` (e.g., `eip155:1`, `solana:5eykt4UsFv8P8NJdTREpY1vzqKqZKvdp`)
- Used for WalletConnect v2 interoperability

### Updated Types (extending en-fr0z)

**`WalletConnection`** — add chain family awareness:
- New field: `supportedChainFamilies: ChainFamily[]`
- New field: `chainProviders: Map<ChainFamily, ChainProviderRef>`
- `supportedChains` becomes computed from `chainProviders`

**`ConnectedAccount`** — add chain family:
- New field: `chainFamily: ChainFamily`

**`AccountId`** — format change:
- New format: `{walletConnectionId}:{chainFamily}:{address}`
- Migration from en-fr0z format: existing EVM-only AccountIds gain the `:evm:` segment

**`WatchAddress`** — add chain family:
- New field: `chainFamily: ChainFamily`

**`IntegrationSource`** — extended:
- Add: `SUI`, `BITCOIN`, `COSMOS`, `APTOS` (alongside existing `EVM`, `SOLANA`)

### What Does NOT Change

- `Balance`, `Value`, `MarketData`, `PriceHistory` — chain-family-agnostic
- `Portfolio`, `AccountPortfolio`, `WalletPortfolio` — from en-fr0z, already account-attributed, chain-family inherits naturally from accounts
- `Transaction` — already has `chainId`; `chainFamily` is derivable from `chainId`
- `ChainRpcConfig`, `RpcProviderConfig` — infrastructure-level, chain-family-agnostic

---

## 8. WalletIntegrationSystem Contract Changes

### Extended WalletConnectionContract (App → WalletIntegration)

Building on the en-fr0z amended contract:

```
WalletConnectionContract {
  // ... all existing en-fr0z commands and queries ...

  // NEW — Multi-chain discovery
  getAvailableWallets(): DiscoveredWallet[]       // replaces en-fr0z's WalletProviderInfo[]
  getWalletCapabilities(walletId: WalletProviderId): ChainFamilyCapability[]

  // NEW — Chain-family-aware connection
  connectWallet(walletId: WalletProviderId, options?: MultiChainConnectionOptions): WalletConnection
  connectChainFamily(connectionId: WalletConnectionId, chainFamily: ChainFamily): ChainProviderRef
  disconnectChainFamily(connectionId: WalletConnectionId, chainFamily: ChainFamily): void

  // NEW — WalletConnect v2
  initiateWalletConnect(options: WalletConnectOptions): WalletConnectSession
  approveWalletConnectSession(session: WalletConnectSession): WalletConnection

  // NEW — Events (supplement en-fr0z events, do not replace them)
  onWalletsDiscovered: Event<DiscoveredWallet[]>
  onChainFamilyConnected: Event<{connectionId: WalletConnectionId, chainFamily: ChainFamily, accounts: ConnectedAccount[]}>
  onChainFamilyDisconnected: Event<{connectionId: WalletConnectionId, chainFamily: ChainFamily}>
  // NOTE: en-fr0z's onWalletConnected and onAccountDiscovered still fire.
  // For a multi-chain connect: onWalletConnected fires once (with the full WalletConnection),
  // onChainFamilyConnected fires once per chain family, and onAccountDiscovered fires once per account.
  // Order: onWalletConnected → onChainFamilyConnected (per family) → onAccountDiscovered (per account).
}

MultiChainConnectionOptions {
  chainFamilies?: ChainFamily[]              // which families to connect (default: all supported)
  label?: string                             // connection label
  autoDetectChains?: boolean                 // auto-detect available chains (default: true)
}

WalletConnectOptions {
  requiredChainFamilies: ChainFamily[]       // must support these
  optionalChainFamilies?: ChainFamily[]      // nice to have
  requiredChains?: ChainId[]                 // specific chains within families
}
```

### Extended WalletIntegrationContract (PortfolioAggregation → WalletIntegration)

Building on the en-fr0z amended contract:

```
WalletIntegrationContract {
  // ... all existing en-fr0z queries and events ...

  // NEW — Chain-family-aware queries
  getTrackedAddressesByChainFamily(chainFamily: ChainFamily): TrackedAddress[]
  getChainFamiliesForConnection(connectionId: WalletConnectionId): ChainFamily[]

  // UPDATED — TrackedAddress now includes chainFamily
  getTrackedAddresses(): TrackedAddress[]     // TrackedAddress extended with chainFamily
}
```

**`TrackedAddress`** — extended:
```
TrackedAddress {
  // existing en-fr0z fields
  accountId: AccountId
  address: string
  walletConnectionId: WalletConnectionId | 'watch'
  providerId: WalletProviderId | 'watch'
  accountLabel: string
  connectionLabel: string
  chainScope: ChainId[]

  // new
  chainFamily: ChainFamily
}
```

---

## 9. Integration Routing

### The Routing Problem

With multi-chain wallets, PortfolioAggregation needs to route address requests to the correct integration bounded context based on chain family:

- EVM addresses → `evm-integration`
- Solana addresses → `sol-integration`
- Sui addresses → future `sui-integration`
- Bitcoin addresses → future `btc-integration`
- Cosmos addresses → future `cosmos-integration`

### Updated PortfolioAggregation Data Flow

```
PortfolioAggregation receives TrackedAddress[]
    │
    ├── Group by chainFamily
    │
    ├── chainFamily=evm → AddressRequest[] → evm-integration.getBalances()
    ├── chainFamily=solana → AddressRequest[] → sol-integration.getBalances()
    ├── chainFamily=sui → AddressRequest[] → (future) sui-integration.getBalances()
    ├── chainFamily=bitcoin → AddressRequest[] → (future) btc-integration.getBalances()
    │
    ├── Combine results
    ├── Attribute to AccountIds
    ├── Deduplicate (same address across connections, within same chain family)
    │
    ▼
    Portfolio with multi-chain, multi-account attribution
```

### AddressRequest Extension

The `AddressRequest` type from en-fr0z gains chain family:

```
AddressRequest {
  accountId: AccountId
  address: string
  chainFamily: ChainFamily          // NEW — which integration to route to
  chainScope: ChainId[]             // chains within the family to query
}
```

### Cross-Chain-Family Deduplication

en-fr0z defined deduplication for same-address-different-AccountIds within the EVM family. Multi-chain adds a new rule:

**Same wallet, different chain families**: Addresses on different chain families are NEVER duplicates (an EVM address and a Solana address from the same Phantom wallet are distinct holdings on distinct chains).

**Same address format, different chains**: Theoretically, the same base58 string could be valid on both Solana and Sui. In practice, the probability is negligible. The `chainFamily` field on `AccountId` and `ConnectedAccount` disambiguates.

**Deduplication only applies within the same chain family**: Same EVM address across two wallet connections → deduplicate for aggregate portfolio (en-fr0z rules). EVM address vs Solana address → never deduplicate.

---

## 10. Impact on Downstream Domains

### PortfolioAggregation

**AccountRegistryService** — extended:
- `registerAccount()` now accepts `TrackedAddress` with `chainFamily`
- `getUniqueAddressesByChain()` → `getUniqueAddressesByChainFamily(family)` — returns deduplicated addresses grouped by chain family for routing to the correct integration
- `resolveAccountIds(address, chainFamily, chainId)` — `chainFamily` parameter added to avoid cross-family collisions

**PortfolioAggregationService** — extended:
- Routing logic: group `TrackedAddress[]` by `chainFamily`, dispatch to correct integration
- Aggregation: merge results from all integrations, keyed by `accountId` (which includes chainFamily)
- Integration registry: maintain a map of `ChainFamily → IntegrationContract` for dispatching

**SyncOrchestrator** — extended:
- Refresh cycles run per-chain-family in parallel (EVM refresh does not block Solana refresh)
- Chain-family-specific error handling (Solana RPC failure doesn't affect EVM data)

### CygnusWealthApp

**Wallet Selection Modal** — updated:
- Discovery results show `DiscoveredWallet` entries with chain family badges
- Multi-chain wallets show a chain family picker before connection
- Single-chain wallets connect directly (no change to existing flow)

**Wallet Management Panel** — updated:
- Each `WalletConnection` shows connected chain families with status
- Per-chain-family "Connect" / "Disconnect" controls
- Accounts grouped by chain family within each wallet connection

**Portfolio Dashboard** — no structural changes:
- en-fr0z's per-account and per-wallet views work naturally with multi-chain accounts
- Asset rows may now span multiple chain families within one wallet view
- Chain family icons appear alongside chain icons in asset attribution

### Integration Domain — Bounded Contexts

**evm-integration**: No change. Receives `AddressRequest[]` with EVM addresses as today.

**sol-integration**: No change. Already accepts address arrays. May need to align with `AddressRequest` pattern from en-fr0z (if not already adopted).

**Future contexts** (sui-integration, btc-integration, cosmos-integration):
- Must implement the `BlockchainIntegrationContract` pattern established by evm-integration
- Accept `AddressRequest[]`, return `AccountBalanceList` with `accountId` attribution
- Follow the anti-corruption layer pattern from the Integration Domain

---

## 11. Migration and Backward Compatibility

### From en-fr0z to en-o8w

en-fr0z introduced the multi-wallet model. en-o8w extends it. The migration path:

1. **AccountId format migration**: Existing `{connectionId}:{address}` format becomes `{connectionId}:evm:{address}`. This is a one-time migration when the schema version increments.

2. **WalletConnection extension**: Existing connections gain `supportedChainFamilies: ['evm']` and a `chainProviders` map with one EVM entry. No data loss.

3. **WatchAddress extension**: Existing watch addresses gain a `chainFamily` field inferred from their address format. Addresses matching EVM format (0x-prefixed, 20 bytes) default to `'evm'`. Addresses matching Solana format (base58, 32-44 chars) default to `'solana'`. If format detection is ambiguous, default to `'evm'` and let the user correct it. en-fr0z's `chainScope: ChainId[]` on watch addresses may contain non-EVM chains — the migration must respect this and derive `chainFamily` from the `chainScope` entries when available.

4. **Schema versioning**:
   - en-fr0z Version 2: Multi-wallet multi-account model
   - en-o8w Version 3: Multi-chain wallet unification model
   - On load, check version and run migration if needed

### Existing Single-Chain Wallets

MetaMask, Rabby, Suiet/Slush, and other single-chain wallets continue to work exactly as before. They produce `WalletConnection` entries with `supportedChainFamilies: ['evm']` (or `['sui']`, etc.) and a single `ChainProviderRef`. The multi-chain machinery is simply not activated for these wallets.

### Discovery Fallback

If EIP-6963 is not supported by a wallet, `window.ethereum` detection remains as fallback. If Wallet Standard is not supported by a Solana wallet, `window.solana` detection remains as fallback. The discovery layer degrades gracefully.

---

## 12. Wallets to Support — Implementation Priority

### Tier 1 — Launch

| Wallet | Chain Families | Discovery | Notes |
|--------|---------------|-----------|-------|
| **Phantom** | EVM, Solana, Bitcoin | EIP-6963 + Wallet Standard + global | Most popular multi-chain wallet |
| **Trust Wallet** | EVM, Solana, Cosmos, Aptos | EIP-6963 + Wallet Standard + global | Widest chain coverage |
| **WalletConnect v2** | Any (namespace-dependent) | Session-based | Universal remote wallet |
| **MetaMask** | EVM only | EIP-6963 | Largest EVM wallet — must continue working |
| **Rabby** | EVM only | EIP-6963 | Popular EVM wallet — must continue working |

### Tier 2 — Fast Follow

| Wallet | Chain Families | Discovery | Notes |
|--------|---------------|-----------|-------|
| **OKX Wallet** | EVM, Solana, Bitcoin | EIP-6963 + Wallet Standard + global | Major multi-chain wallet |
| **Backpack** | EVM, Solana | EIP-6963 + Wallet Standard | Growing multi-chain wallet |
| **Crypto.com DeFi** | EVM, Cosmos | EIP-6963 + global | chainType toggle model |

### Tier 3 — Future

| Wallet | Chain Families | Notes |
|--------|---------------|-------|
| **Suiet/Slush** | Sui only | Wallet Standard |
| **Keplr** | Cosmos only | Global injection |
| **Xverse** | Bitcoin only | Sats Connect |

### Adding New Wallets

Adding support for a new multi-chain wallet requires:
1. Add entry to the correlation registry (RDNS → provider ID mapping)
2. Implement any wallet-specific adapter overrides (if the wallet deviates from standards)
3. Add provider metadata (icon, display name)
4. No changes to core architecture, contracts, or data model

---

## 13. Security Considerations

### Read-Only Guarantees

All en-fr0z security guarantees carry forward:
- **No private key access** across any chain family
- **No transaction signing** — we never request signing methods
- **Minimal permissions** — only account/address read permissions
- **Session data encryption** — all persisted data encrypted via EncryptionService

### Chain-Family-Specific Security

**EVM**: Only `eth_requestAccounts` / `eth_accounts`. No `eth_sendTransaction`, `personal_sign`, or other signing methods. EIP-6963 removes the `window.ethereum` spoofing risk (each provider is independently verified by RDNS).

**Solana (Wallet Standard)**: Only `standard:connect`. No `solana:signAndSendTransaction`, `solana:signMessage`, or `solana:signTransaction`.

**Bitcoin**: Only `requestAccounts` or equivalent read-only methods. No `signPsbt`, `sendBitcoin`, or other transaction methods.

**WalletConnect v2**: Session proposal includes ONLY read-only methods in the `methods` field. If a remote wallet somehow requests signing, the request is rejected at the application layer.

### Provider Integrity

- **EIP-6963 RDNS validation**: RDNS values are checked against the correlation registry. Unknown RDNS values are treated as EVM-only with a generic display.
- **Wallet Standard name validation**: Same pattern — unknown names are treated as single-chain.
- **No auto-execution**: Wallet providers are never auto-connected. The user must explicitly select and approve each connection.

---

## 14. Performance Considerations

### Scale Targets (extending en-fr0z)

| Metric | en-fr0z Target | en-o8w Target |
|--------|---------------|---------------|
| Wallet connections | Up to 10 | Up to 10 |
| Chain families per wallet | 1 (EVM only) | Up to 4 |
| Accounts per connection | Up to 20 | Up to 20 (across all chain families) |
| Total tracked accounts | Up to 100 | Up to 100 |
| Unique addresses (after dedup) | Up to 50 | Up to 50 per chain family |
| Chains per address | Up to 8 (EVM) | Up to 8 per chain family |

### Discovery Performance

- EIP-6963 discovery: < 100ms (event-based, near-instant)
- Wallet Standard discovery: < 100ms (registration-based)
- Global injection probes: < 50ms (synchronous property checks)
- Correlation phase: < 10ms (static map lookups)
- Total discovery: < 200ms

### Connection Performance

Per-chain-family connect calls are made in parallel (not sequential):
- EVM connect: < 3 seconds (en-fr0z SLA, unchanged)
- Solana connect: < 2 seconds
- Bitcoin connect: < 2 seconds
- Total multi-chain connect (parallel): < 3 seconds (bounded by slowest family)

### Memory Budget (extending en-fr0z)

- `ChainProviderRef` per chain family: ~0.5KB
- Multi-chain `WalletConnection` with 4 chain families: ~4KB (vs 2KB for EVM-only)
- Correlation registry: ~2KB (static, shared)
- Total overhead of multi-chain extension: < 5KB per connection
- Well within en-fr0z's 10MB local storage budget

---

## 15. Interaction with Existing Directives

### en-fr0z (Multi-Wallet Multi-Account Architecture)

This directive is a **superset extension** of en-fr0z. Every type, contract, and pattern from en-fr0z remains valid. This directive adds:
- `ChainFamily` dimension to identity model
- Multi-chain discovery layer
- Chain-family-specific adapters
- WalletConnect v2 multi-namespace support

en-fr0z's EVM-specific details (EIP-1193, `eth_requestAccounts`, `accountsChanged`) are preserved as the `EvmChainFamilyAdapter` implementation.

### en-p0ha (WebSocket-First Architecture)

- Subscription events gain `chainFamily` context alongside existing `chainId` and `accountIds`
- SubscriptionOrchestrator routes events to the correct chain-family integration
- No fundamental architecture change

### en-tfkn / en-25w5 (RPC Provider Strategy)

- RPC infrastructure is per-chain, not per-wallet or per-chain-family
- Multi-chain wallets generate more addresses to track, but RPC routing is unchanged
- Solana/Sui/Bitcoin RPC strategies will follow the same fallback chain pattern

### hq-onegb / hq-su5pt (CEX Integrations)

- No change. CEX integrations authenticate independently of wallet connections
- A multi-chain wallet connection does not affect CEX data flows

---

## 16. Testing Scenarios

### Multi-Chain Discovery

- Install Phantom and MetaMask. Discovery should list: Phantom (EVM, Solana, Bitcoin), MetaMask (EVM). Phantom appears once, not three times.
- Install only MetaMask. Discovery should list: MetaMask (EVM). No multi-chain indicators.
- Install Trust Wallet. Discovery should list: Trust Wallet (EVM, Solana, Cosmos, Aptos) — all chain families detected via their respective protocols.
- Install unknown EVM wallet (EIP-6963 only). Discovery should list it as EVM-only with its provider name and icon.

### Multi-Chain Connection

- Connect Phantom, select EVM + Solana. One WalletConnection created with accounts from both chain families. AccountIds contain `evm:` and `solana:` segments.
- Connect Phantom, select only EVM. One WalletConnection with only EVM accounts. Later click "Connect Solana" — Solana accounts added to same connection.
- Connect Trust Wallet, EVM succeeds but Cosmos fails. WalletConnection created with EVM accounts. Cosmos shows as "not connected" with retry option.
- Connect via WalletConnect v2, approve EVM + Solana. One WalletConnection with accounts from both namespaces. Peer metadata maps to wallet name if known.

### Multi-Chain Portfolio

- Connect Phantom with EVM + Solana. Portfolio shows ETH tokens and SOL tokens, all attributed to "Phantom" wallet. Per-account view shows EVM account separately from Solana account.
- Same Solana address in Phantom (via wallet connect) and as a watch address. Aggregate portfolio deduplicates. Per-account view shows both.
- Connect MetaMask (EVM) and Phantom (EVM + Solana). If same EVM address exists in both, it is deduplicated in aggregate view (en-fr0z rules). Solana addresses are unique to Phantom.

### Backward Compatibility

- Existing single-chain wallet connections (pre-en-o8w) migrate successfully to Version 3 schema.
- AccountId format `{connectionId}:{address}` automatically migrates to `{connectionId}:evm:{address}`.
- All existing portfolio views continue to work without changes.

---

## 17. Open Questions

### Bitcoin Address Discovery

Bitcoin wallets use different address types (P2PKH, P2SH, P2WPKH, P2WSH, P2TR). Phantom Bitcoin exposes one address type at a time. Should we:
- Track only the address the wallet exposes (simplest, consistent with read-only philosophy)
- Allow users to add other address types for the same wallet manually (via watch address)

**Recommendation**: Track only the exposed address. Users can add other address types as watch addresses if needed.

### Cosmos Chain Selection

Cosmos is not one chain — it's an ecosystem of chains (Cosmos Hub, Osmosis, Celestia, etc.) each requiring separate `enable()` calls via the Keplr interface. Should we:
- Enable all supported Cosmos chains automatically on connection
- Let users select which Cosmos chains to track (similar to EVM chain scope)

**Recommendation**: Let users select Cosmos chains, defaulting to Cosmos Hub. This prevents unnecessary RPC calls to chains the user doesn't use.

### Future Chain Family Extensibility

The `ChainFamily` enum is deliberately closed (not a free-form string) to ensure type safety. Adding new chain families (e.g., `'polkadot'`, `'tron'`, `'near'`) requires:
- Adding the value to the `ChainFamily` enum
- Implementing a `ChainFamilyAdapter`
- Adding correlation registry entries
- Adding an integration bounded context

This is intentional — each chain family is a meaningful architectural boundary.

---

## Cross-Domain Contract Impact Summary

| Contract | Change from en-fr0z | Details |
|----------|---------------------|---------|
| App → Wallet Integration | **Extended** | Discovery returns `DiscoveredWallet` with chain family capabilities; connection options include chain family selection |
| Portfolio Aggregation → Wallet Integration | **Extended** | `TrackedAddress` includes `chainFamily`; new `getTrackedAddressesByChainFamily()` query |
| Portfolio Aggregation → Blockchain Integrations | **Extended** | `AddressRequest` includes `chainFamily` for routing; routing logic dispatches to correct integration |
| App → Portfolio Aggregation | **No change** | en-fr0z's per-account/per-wallet queries work naturally with multi-chain accounts |
| Portfolio Aggregation → Asset Valuator | **No change** | Asset valuation is chain-family-agnostic |
| Portfolio Aggregation → CEX Integrations | **No change** | CEX integrations are independent of wallet connections |
| Data Models | **Extended** | New `ChainFamily` type, `Caip2ChainId` type; existing types gain `chainFamily` field |

---

## Data Flow — Updated

```
User clicks "Connect Wallet" in CygnusWealthApp
    │
    ▼
WalletDiscoveryService
    ├── EIP-6963 announcements (EVM providers)
    ├── Wallet Standard registrations (Solana/Sui/Aptos providers)
    ├── Global injection probes (Bitcoin/Cosmos providers)
    ├── Correlation phase (group by canonical WalletProviderId)
    ├── Produces DiscoveredWallet[] with chain family capabilities
    │
    ▼
User selects wallet and chain families
    │
    ▼
WalletIntegrationSystem
    ├── Per-chain-family connect (in parallel):
    │     ├── EvmChainFamilyAdapter.connect(eip1193Provider) → EVM accounts
    │     ├── SolanaChainFamilyAdapter.connect(walletStandardWallet) → Solana accounts
    │     ├── BitcoinChainFamilyAdapter.connect(bitcoinProvider) → Bitcoin accounts
    │     └── CosmosChainFamilyAdapter.connect(cosmosProvider) → Cosmos accounts
    ├── Creates single WalletConnection with multi-chain accounts
    ├── Persists to encrypted local storage
    ├── Emits onWalletConnected, onAccountDiscovered (per account)
    │
    ▼
PortfolioAggregation (AccountRegistryService)
    ├── Registers TrackedAddresses (now with chainFamily)
    ├── Deduplicates within each chain family
    ├── Groups by chain family for integration routing
    │
    ├── chainFamily=evm ──► evm-integration.getBalances(addressRequests)
    ├── chainFamily=solana ──► sol-integration.getBalances(addressRequests)
    ├── chainFamily=bitcoin ──► btc-integration.getBalances(addressRequests) [future]
    ├── chainFamily=cosmos ──► cosmos-integration.getBalances(addressRequests) [future]
    │
    ├── Combines results from all integrations
    ├── Attributes to AccountIds (which include chainFamily)
    ├── Builds Portfolio with cross-chain account/wallet breakdowns
    │
    ▼
CygnusWealthApp
    ├── Shows unified portfolio across all chain families
    ├── Per-wallet view: "Phantom" shows ETH, SOL, and BTC holdings together
    ├── Per-account view: each chain-family account shown individually
    ├── Aggregate view: all holdings across all wallets and chains
    │
    ▼
User sees unified multi-chain portfolio with clear wallet attribution
```
