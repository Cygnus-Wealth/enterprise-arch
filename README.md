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

Each bounded context lives in its own repository under the `cygnus-wealth` GitHub organization:

```
cygnus-wealth/                        # GitHub Organization
├── data-models/                      # Shared contracts and unified data structures
├── portfolio-aggregation/            # Portfolio orchestration and aggregation logic
├── cygnus-wealth-core/              # User interface and user experience
├── asset-valuator/                  # Pricing and valuation services
├── evm-integration/                 # EVM blockchain integration
├── sol-integration/                 # Solana blockchain integration
└── robinhood-integration/           # Traditional finance integration
```

See [Repository Organization Strategy](./repository-organization.md) for detailed GitHub structure.

## Architecture Layers

### 1. Contract Layer
- **data-models**: Unified data structures that normalize diverse sources
- Acts as the ubiquitous language across all domains
- Ensures type safety and consistency
- Provides the canonical representation for all assets

### 2. Integration Layer
- **evm-integration**: EVM blockchain data retrieval and wallet tracking
- **sol-integration**: Solana blockchain data retrieval and wallet tracking
- **robinhood-integration**: Traditional finance data retrieval
- Each integration handles its own connection management and data fetching

### 3. Service Layer
- **portfolio-aggregation**: Orchestrates data collection from all integrations
- **asset-valuator**: Provides pricing and valuation services

### 4. Application Layer
- **cygnus-wealth-core**: User interface, interaction, and configuration
- Manages user preferences including which addresses to track
- Handles visualization and user experience

## Domain Documents

- [Data Models Domain](./domains/data-models.md) - Unified data structures
- [Portfolio Aggregation Domain](./domains/portfolio-aggregation.md) - Orchestration logic
- [Core Application Domain](./domains/cygnus-wealth-core.md) - User interface
- [Asset Valuator Domain](./domains/asset-valuator.md) - Pricing services
- [EVM Integration Domain](./domains/evm-integration.md) - Ethereum/EVM support
- [Solana Integration Domain](./domains/sol-integration.md) - Solana support
- [Robinhood Integration Domain](./domains/robinhood-integration.md) - TradFi support
- [Domain Contracts](./contracts.md) - Inter-domain communication contracts

## Repository Organization

- [Repository Organization Strategy](./repository-organization.md) - GitHub organization and repository structure

## Communication Patterns

### Event Flow
1. User interactions trigger events in **cygnus-wealth-core**
2. Core delegates to **portfolio-aggregation** for data orchestration
3. Portfolio aggregation coordinates calls to integration domains
4. Integration domains fetch data from external sources
5. Data is transformed to unified **data-models** format
6. **asset-valuator** enriches data with pricing information
7. Portfolio aggregation combines and deduplicates data
8. Core receives aggregated portfolio and presents to user

### Data Flow
```
External Sources → Integration Domains → Portfolio Aggregation → Core → User
                            ↓                     ↓
                      Data Models          Asset Valuator
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