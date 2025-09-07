# Portfolio Domain

## Overview

The Portfolio Domain is the core business domain of CygnusWealth, responsible for portfolio management, aggregation, and valuation. This is where the primary value proposition lives - transforming disparate asset data into unified, actionable portfolio insights.

## Strategic Importance

This is the **Core Domain** in DDD terms - it represents the competitive advantage and unique business value of CygnusWealth. It contains the complex business logic for aggregating multi-source portfolios, calculating valuations, and providing portfolio analytics.

## Bounded Contexts

### 1. Portfolio Aggregation
- **Responsibility**: Orchestration and aggregation of data from all sources
- **Repository**: `portfolio-aggregation`
- **Documentation**: [Portfolio Aggregation Context](./bounded-contexts/portfolio-aggregation.md)

### 2. Asset Valuator
- **Responsibility**: Pricing, valuation, and currency conversion
- **Repository**: `asset-valuator`
- **Documentation**: [Asset Valuator Context](./bounded-contexts/asset-valuator.md)

## Domain Principles

### 1. Business Logic Centralization
All portfolio-related business logic lives here:
- Aggregation strategies
- Deduplication rules
- Valuation calculations
- Portfolio metrics computation

### 2. Source Agnostic
The domain works with normalized data:
- Doesn't know about specific blockchains
- Unaware of integration details
- Focuses on portfolio concepts
- Uses unified data models

### 3. Orchestration Excellence
Coordinates complex workflows:
- Parallel data fetching
- Multi-source reconciliation
- Partial failure handling
- Incremental updates

### 4. Performance Critical
This domain is performance-sensitive:
- Efficient aggregation algorithms
- Smart caching strategies
- Optimized calculations
- Minimal latency

## Core Capabilities

### Portfolio Management
- **Aggregation**: Combine assets from multiple sources
- **Deduplication**: Identify and merge duplicate assets
- **Reconciliation**: Handle discrepancies between sources
- **Calculation**: Compute portfolio metrics and analytics

### Valuation Services
- **Real-time Pricing**: Current market values
- **Historical Values**: Track portfolio over time
- **Currency Conversion**: Multi-currency support
- **Custom Valuations**: User-defined asset values

### Analytics
- **Performance Tracking**: Returns and growth metrics
- **Asset Allocation**: Distribution analysis
- **Risk Assessment**: Portfolio risk metrics
- **Trend Analysis**: Historical patterns

## Business Rules

### Aggregation Rules
1. **Precedence**: On-chain data preferred over CEX data
2. **Freshness**: Use most recent data when sources conflict
3. **Completeness**: Return partial results with clear indicators
4. **Accuracy**: Validate data consistency across sources

### Valuation Rules
1. **Price Sources**: Aggregate from multiple sources when possible
2. **Fallback**: Use last known good price when current unavailable
3. **Custom Assets**: Allow user-defined valuations
4. **Currency**: Always store base value and converted value

### Deduplication Logic
1. **Asset Matching**: By contract address, symbol, or identifier
2. **Quantity Handling**: Sum quantities from different sources
3. **Metadata**: Preserve most complete metadata
4. **Conflict Resolution**: Clear rules for discrepancies

## Technical Architecture

### Service Boundaries
Clear interfaces between contexts:
- Portfolio Aggregation orchestrates
- Asset Valuator provides pricing
- Neither directly accesses integrations

### Data Flow
```
Integration Data → Portfolio Aggregation → Asset Valuator → Enriched Portfolio
```

### Caching Strategy
- **Portfolio Cache**: 5-minute TTL for complete portfolios
- **Price Cache**: 60-second TTL for active prices
- **Calculation Cache**: Cache computed metrics
- **Invalidation**: Smart invalidation on updates

## Quality Attributes

### Reliability
- 99.9% uptime target
- Graceful degradation
- Partial result handling
- Error recovery

### Performance
- Sub-3 second full portfolio aggregation
- Real-time price updates
- Efficient batch operations
- Optimized calculations

### Scalability
- Handle 1000+ assets per portfolio
- Support 100+ concurrent users
- Efficient resource usage
- Horizontal scaling ready

## Dependencies

### Internal Dependencies
- `data-models`: Unified data structures
- Integration contexts: For raw data

### External Dependencies
- Price aggregator APIs
- Market data services
- Currency exchange rates

## Testing Strategy

### Unit Testing
- Business logic validation
- Calculation accuracy
- Rule enforcement
- Edge cases

### Integration Testing
- Multi-source aggregation
- End-to-end workflows
- Performance benchmarks
- Failure scenarios

## Future Enhancements

### Planned Features
- Advanced portfolio analytics
- Machine learning predictions
- Risk modeling
- Tax optimization
- Rebalancing suggestions

### Technical Improvements
- GraphQL API
- Real-time subscriptions
- Advanced caching
- Distributed processing

## Domain Events

### Published Events
- Portfolio Updated
- Asset Price Changed
- Aggregation Completed
- Valuation Calculated

### Event Handling
- Asynchronous processing
- Event sourcing capability
- Audit trail
- Replay functionality

## Governance

### Domain Ownership
The Portfolio Domain team owns:
- Business logic implementation
- Performance optimization
- Feature development
- Quality assurance

### Decision Making
- Business rules defined by product team
- Technical implementation by domain team
- Architecture decisions require review
- Performance targets agreed with stakeholders