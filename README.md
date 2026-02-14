# CygnusWealth Enterprise Architecture

## Overview

CygnusWealth is a privacy-first, decentralized portfolio aggregation system built with Domain-Driven Design principles. The architecture emphasizes client-side sovereignty, read-only integrations, and modular domain boundaries.

## Core Principles

1. **Client-Side Sovereignty**: All processing happens in the browser, no server-side data storage
2. **Read-Only Operations**: System never handles private keys or transaction signing
3. **Domain Isolation**: Each domain has clear boundaries and responsibilities
4. **Type Safety**: Shared data models ensure consistency across domains
5. **Privacy First**: No telemetry, analytics, or external data sharing

## Domain Structure

CygnusWealth is organized into four strategic domains, each containing one or more bounded contexts:

```
Domains/
├── Contract/                         # Shared contracts and data models
│   └── data-models                  # Unified data structures
├── Portfolio/                        # Core business domain
│   ├── portfolio-aggregation        # Orchestration and aggregation
│   └── asset-valuator              # Pricing and valuation
├── Integration/                      # External data acquisition
│   ├── wallet-integration-system   # Wallet connections
│   ├── evm-integration             # Ethereum/EVM blockchains
│   ├── sol-integration             # Solana blockchain
│   └── robinhood-integration       # Traditional finance
└── Experience/                       # User interfaces
    └── cygnus-wealth-app           # Main web application
```

Each bounded context lives in its own repository under the `cygnus-wealth` GitHub organization.
See [Repository Organization Strategy](./repository-organization.md) for detailed GitHub structure.

## Architecture Layers

### 1. Contract Layer
- **data-models**: Unified data structures that normalize diverse sources
- Acts as the ubiquitous language across all domains
- Ensures type safety and consistency
- Provides the canonical representation for all assets

### 2. Integration Layer (Integration Domain)
- **wallet-integration-system**: Manages wallet provider connections
- **evm-integration**: EVM blockchain data retrieval and wallet tracking
- **sol-integration**: Solana blockchain data retrieval and wallet tracking
- **robinhood-integration**: Traditional finance data retrieval
- Each integration handles its own connection management and data fetching

### 3. Service Layer (Portfolio Domain)
- **portfolio-aggregation**: Orchestrates data collection from all integrations
- **asset-valuator**: Provides pricing and valuation services

### 4. Application Layer (Experience Domain)
- **cygnus-wealth-app**: User interface, interaction, and configuration
- Manages user preferences including which addresses to track
- Handles visualization and user experience

## Domain Documentation

### Strategic Domains
- [Contract Domain](./domains/contract/README.md) - Shared language and contracts
- [Portfolio Domain](./domains/portfolio/README.md) - Core business logic
- [Integration Domain](./domains/integration/README.md) - External data acquisition
- [Experience Domain](./domains/experience/README.md) - User interfaces

### Bounded Contexts
#### Contract Domain
- [Data Models](./domains/contract/bounded-contexts/data-models.md) - Unified data structures

#### Portfolio Domain
- [Portfolio Aggregation](./domains/portfolio/bounded-contexts/portfolio-aggregation.md) - Orchestration logic
- [Asset Valuator](./domains/portfolio/bounded-contexts/asset-valuator.md) - Pricing services

#### Integration Domain
- [Wallet Integration System](./domains/integration/bounded-contexts/wallet-integration-system.md) - Wallet connections
- [EVM Integration](./domains/integration/bounded-contexts/evm-integration.md) - Ethereum/EVM support
- [Solana Integration](./domains/integration/bounded-contexts/sol-integration.md) - Solana support
- [Robinhood Integration](./domains/integration/bounded-contexts/robinhood-integration.md) - TradFi support

#### Experience Domain
- [CygnusWealth App](./domains/experience/bounded-contexts/cygnus-wealth-app.md) - Web application

### Architecture Documents
- [Domain Contracts](./contracts.md) - Inter-domain communication contracts
- [E2E Testing Strategy](./e2e-testing-strategy.md) - Enterprise-wide E2E testing policy per bounded context

## Repository Organization

- [Repository Organization Strategy](./repository-organization.md) - GitHub organization and repository structure

## Communication Patterns

### Event Flow
1. User interactions trigger events in **cygnus-wealth-app**
2. App delegates to **portfolio-aggregation** for data orchestration
3. **wallet-integration-system** provides connected addresses
4. Portfolio aggregation coordinates calls to blockchain integrations
5. Integration domains fetch data from external sources
6. Data is transformed to unified **data-models** format
7. **asset-valuator** enriches data with pricing information
8. Portfolio aggregation combines and deduplicates data
9. App receives aggregated portfolio and presents to user

### Data Flow
```
External Sources → Integration Domain → Portfolio Domain → Experience Domain → User
                            ↓                    ↓
                    Contract Domain      Asset Valuator
                    (Data Models)
```

## Technology Stack

- **TypeScript**: Primary language across all domains
- **React 19**: UI framework (core application)
- **Vite**: Build tooling
- **Vitest**: Testing framework
- **Web3 Libraries**: ethers, @solana/web3.js
- **State Management**: Zustand, React Context

## Security Considerations

1. **No Private Keys**: System never accesses or stores private keys
2. **Read-Only**: All blockchain interactions are read-only
3. **Local Storage**: Encrypted local storage for user preferences
4. **IPFS Backup**: Optional decentralized backup without central servers
5. **Client Validation**: All data validation happens client-side

## Development Guidelines

1. **Domain Boundaries**: Respect domain boundaries, avoid cross-domain dependencies
2. **Contract First**: Define data models before implementation
3. **Type Safety**: Leverage TypeScript for compile-time safety
4. **Testing**: Unit tests for domains, integration tests for contracts
5. **Documentation**: Keep domain documentation up-to-date

## Future Considerations

- Additional blockchain integrations (Bitcoin, Cosmos, etc.)
- More CEX integrations (Coinbase, Binance, etc.)
- Advanced portfolio analytics
- Multi-signature wallet support
- Hardware wallet integration