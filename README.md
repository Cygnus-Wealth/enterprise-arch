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

```
cygnus-wealth/
├── data-models/              # Shared contracts and data structures
├── cygnus-wealth-core/       # Main UI and orchestration layer
├── asset-valuator/           # Pricing and valuation services
├── wallet-integration-system/# Multi-chain wallet connections
├── evm-integration/          # EVM blockchain integration
├── sol-integration/          # Solana blockchain integration
└── robinhood-integration/    # Traditional finance integration
```

## Architecture Layers

### 1. Contract Layer (data-models)
- Defines all shared interfaces and data structures
- Acts as the ubiquitous language across domains
- Ensures type safety and consistency

### 2. Integration Layer
- **wallet-integration-system**: Manages wallet connections
- **evm-integration**: EVM blockchain data retrieval
- **sol-integration**: Solana blockchain data retrieval
- **robinhood-integration**: Traditional finance data retrieval

### 3. Service Layer
- **asset-valuator**: Provides pricing and valuation services

### 4. Application Layer
- **cygnus-wealth-core**: User interface and orchestration

## Domain Documents

- [Data Models Domain](./domains/data-models.md) - Shared contracts and types
- [Core Application Domain](./domains/cygnus-wealth-core.md) - Main UI and orchestration
- [Asset Valuator Domain](./domains/asset-valuator.md) - Pricing services
- [Wallet Integration Domain](./domains/wallet-integration-system.md) - Wallet connections
- [EVM Integration Domain](./domains/evm-integration.md) - Ethereum/EVM support
- [Solana Integration Domain](./domains/sol-integration.md) - Solana support
- [Robinhood Integration Domain](./domains/robinhood-integration.md) - TradFi support
- [Domain Contracts](./contracts.md) - Inter-domain communication contracts

## Communication Patterns

### Event Flow
1. User interactions trigger events in **cygnus-wealth-core**
2. Core orchestrates calls to integration domains
3. Integration domains fetch data from external sources
4. Data is transformed to standard **data-models** format
5. **asset-valuator** enriches data with pricing
6. Core aggregates and presents data to user

### Data Flow
```
External Sources → Integration Domains → Data Models → Core → User Interface
                                              ↓
                                        Asset Valuator
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