# CygnusWealth App

## Bounded Context Overview

The CygnusWealth App is the main web application that provides the user interface and user experience for the portfolio management system. It focuses on presentation, user interaction, and managing user preferences while delegating data orchestration to the portfolio-aggregation domain.

## Core Responsibilities

### Primary Functions
1. **User Interface**: React-based responsive web application
2. **User Experience**: Navigation, interactions, and visual feedback
3. **State Management**: UI state and user preferences with Zustand
4. **User Configuration**: Managing which addresses and accounts to track
5. **User Authentication**: Local authentication and encrypted storage
6. **Portfolio Visualization**: Charts, tables, and analytics display
7. **IPFS Integration**: Decentralized backup and sync capabilities

### Domain Boundaries
- **Owns**: UI components, user interactions, user configuration, local storage
- **Does NOT Own**: Portfolio aggregation logic, direct blockchain interactions, price fetching

## Architecture Patterns

### Component Structure
- **Components**: Reusable UI components organized by feature (Portfolio, Charts, Common)
- **Hooks**: Custom React hooks for state and side effects
- **Services**: Integration with domain services
- **Stores**: Zustand state management
- **Utils**: Helper functions and utilities

### State Management

#### UI State Store
- Current view and navigation state
- Modal and dialog visibility
- Loading and error states
- User notifications

#### User Configuration Store
- Tracked addresses per blockchain
- API keys for CEX integrations
- Display preferences and settings
- Authentication state

#### Cache Store
- Cached portfolio data from aggregation service
- Last refresh timestamps
- Offline data availability

## Service Integration

### Portfolio Service Integration
The core delegates all data aggregation to the portfolio-aggregation domain:

1. **Request Portfolio**: Core requests aggregated data from portfolio-aggregation service
2. **Display Data**: Transforms aggregated data for UI presentation
3. **Handle Updates**: Subscribes to portfolio update events
4. **Manage Configuration**: Sends user configuration changes to aggregation service

## Key Components

### Core Services
1. **ConfigurationManager**: Manages user's tracked addresses and API keys
2. **PresentationService**: Transforms domain data for UI display
3. **DataSyncService**: Handles IPFS backup and sync
4. **CacheManager**: Local caching for performance
5. **EncryptionService**: Client-side encryption for sensitive data

### UI Components

#### Main Application Structure
- App Shell with authentication
- Dashboard layout
- Portfolio views and visualizations
- Configuration panels for addresses and integrations
- Settings and preferences management

## External Dependencies

### Required Domains
- **data-models**: All data type definitions
- **portfolio-aggregation**: Portfolio orchestration and aggregation
- **asset-valuator**: Price and valuation data (indirect via aggregation)

### Third-Party Libraries
- **React 19**: UI framework
- **Chakra UI v3**: Component library
- **Recharts**: Data visualization
- **Zustand**: State management
- **IPFS HTTP Client**: Decentralized storage

## User Workflows

### Primary User Journeys

1. **Initial Setup**
   ```
   User → Authentication → Create Portfolio → Connect Integrations
   ```

2. **Portfolio Viewing**
   ```
   User → Dashboard → Select Portfolio → View Analytics
   ```

3. **Adding Integrations**
   ```
   User → Integration Manager → Select Type → Connect → Sync Data
   ```

4. **Data Refresh**
   ```
   User → Refresh Button → Orchestrator → All Integrations → Update UI
   ```

## Data Flow

### Simplified Pipeline
1. User requests portfolio refresh in UI
2. Core calls portfolio-aggregation service
3. Portfolio-aggregation handles all orchestration
4. Aggregated portfolio returned to core
5. Core transforms data for presentation
6. UI updates with new data

## Security & Privacy

### Client-Side Security
1. **No Server Storage**: All data stored locally in browser
2. **Encryption**: AES-256 encryption for sensitive data
3. **Password Protection**: Local authentication required
4. **No Telemetry**: Zero tracking or analytics
5. **Read-Only**: Never handles private keys

### Data Protection
- Encryption key derived from user password
- Auto-lock after period of inactivity (default: 15 minutes)
- Optional data clearing on logout
- IPFS backup encryption enabled by default

## Performance Optimization

### Strategies
1. **Lazy Loading**: Load integrations on demand
2. **Memoization**: Cache computed values
3. **Virtual Scrolling**: For large transaction lists
4. **Web Workers**: Heavy computations off main thread
5. **Progressive Enhancement**: Core features work without all integrations

### Caching Strategy
- Portfolio data: 5-minute TTL
- Price data: 1-minute TTL
- Transaction history: 10-minute TTL
- Local storage: 10MB limit

## Error Handling

### User-Facing Errors
1. **Integration Failures**: Show partial data with warning
2. **Network Issues**: Offline mode with cached data
3. **Invalid Data**: Validation errors with helpful messages
4. **Rate Limiting**: Queue and retry with user notification

### Recovery Strategies
- Automatic retry with exponential backoff
- Fallback to cached data when available
- Graceful degradation of features
- Clear error messages with actionable steps

## Configuration

### User Preferences
- Theme selection (light/dark/system)
- Display currency preference
- Auto-refresh intervals
- Notification settings
- Display customization options

### Environment Configuration
- IPFS gateway selection
- Retry limits for failed requests
- Debug mode toggle
- Feature flag management

## Testing Strategy

### Test Coverage
1. **Unit Tests**: Components, hooks, utilities
2. **Integration Tests**: Domain service interactions
3. **E2E Tests**: User workflows with Playwright
4. **Visual Tests**: Component screenshots

### Key Test Scenarios
- Portfolio aggregation with multiple sources
- Offline mode functionality
- Error recovery flows
- Data encryption/decryption
- IPFS backup and restore

## Future Enhancements

### Planned Features
1. **Mobile App**: React Native implementation
2. **Advanced Analytics**: DeFi yield tracking, impermanent loss
3. **Tax Reporting**: Transaction categorization and reports
4. **Multi-User Support**: Family/team portfolios
5. **Automated Strategies**: Rebalancing suggestions

### Technical Improvements
- Server-side rendering for SEO
- PWA capabilities for offline use
- WebAssembly for performance-critical code
- P2P sync without IPFS dependency