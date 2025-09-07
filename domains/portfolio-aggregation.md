# Portfolio Aggregation Domain

## Overview

The Portfolio Aggregation domain is responsible for orchestrating data collection from all integration sources and combining them into a unified portfolio view. It serves as the central coordination point between the UI layer and the various data sources.

## Core Responsibilities

1. **Data Orchestration**: Coordinates parallel data fetching from multiple integration domains
2. **Portfolio Composition**: Combines assets from different sources into a unified portfolio
3. **Deduplication**: Identifies and merges duplicate assets across sources
4. **Reconciliation**: Handles discrepancies between different data sources
5. **Caching Strategy**: Implements intelligent caching to minimize external API calls
6. **Error Handling**: Provides graceful degradation when some sources are unavailable

## Domain Boundaries

### What This Domain Owns
- Portfolio aggregation logic and strategies
- Cross-source data reconciliation rules
- Portfolio calculation algorithms
- Orchestration workflows
- Aggregation-specific caching policies

### What This Domain Doesn't Own
- Direct blockchain/API connections (owned by integration domains)
- UI components and user interactions (owned by core)
- Price data and valuation logic (owned by asset-valuator)
- Data model definitions (owned by data-models)

## Dependencies

### Depends On
- **data-models**: For unified data structures
- **evm-integration**: For EVM blockchain data
- **sol-integration**: For Solana blockchain data
- **robinhood-integration**: For traditional finance data
- **asset-valuator**: For pricing and valuation

### Depended On By
- **cygnus-wealth-core**: Uses aggregation services for portfolio data

## Key Concepts

### Portfolio
The complete aggregated view of a user's assets across all sources, including:
- Cryptocurrency holdings
- Traditional finance positions
- NFT collections
- DeFi positions
- Total valuations

### Aggregation Strategy
Defines how data from multiple sources is combined:
- **Parallel Fetching**: All sources queried simultaneously
- **Fallback Chains**: Alternative sources when primary fails
- **Partial Results**: Return available data even if some sources fail

### Reconciliation Rules
How to handle the same asset from multiple sources:
- Prefer on-chain data over CEX data
- Use most recently updated source for pricing
- Sum quantities across sources for total holdings

## Service Interface

### Primary Services

#### Portfolio Aggregation Service
- Fetches data from all configured sources
- Applies reconciliation and deduplication
- Returns unified portfolio

#### Address Registry Service
- Manages which addresses to track per chain
- Stores user labels and metadata
- Provides address discovery capabilities

#### Sync Orchestrator
- Coordinates refresh cycles
- Manages rate limiting across sources
- Implements retry logic with exponential backoff

## Integration Patterns

### Command Pattern
All aggregation requests follow a command pattern:
- Aggregate Portfolio Command
- Refresh Source Command
- Add Address Command

### Observer Pattern
Notifies subscribers of portfolio updates:
- Portfolio Updated Event
- Source Sync Completed Event
- Error Occurred Event

## Error Handling

### Partial Failure Strategy
- Continue aggregation even if individual sources fail
- Mark failed sources in response metadata
- Cache last known good data for failed sources

### Circuit Breaker
- Track failure rates per integration
- Temporarily skip failing integrations
- Automatic recovery attempts with backoff

## Performance Considerations

### Caching Layers
1. **Memory Cache**: Hot data for current session
2. **Local Storage**: Persistent cache across sessions
3. **Stale-While-Revalidate**: Serve cached data while fetching updates

### Optimization Strategies
- Parallel execution of independent operations
- Lazy loading of detailed data
- Incremental updates rather than full refreshes
- Delta synchronization where supported

## Security Considerations

- Never stores private keys or credentials
- All sensitive data encrypted before caching
- Read-only operations only
- Validates all data from external sources

## Future Enhancements

- Machine learning for duplicate detection
- Advanced portfolio analytics
- Historical portfolio tracking
- Multi-portfolio support
- Custom aggregation strategies