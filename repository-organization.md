# CygnusWealth Repository Organization Strategy

## Overview

CygnusWealth uses a multi-repository architecture with GitHub Organizations and topic tags to maintain clean domain boundaries while enabling discoverability and consistent management across all bounded contexts.

## GitHub Organization Structure

### Organization: `cygnus-wealth`

All repositories live under a single GitHub organization to provide:
- Unified namespace and URL structure
- Organization-level access control
- Shared secrets and CI/CD configuration
- Centralized billing and management

## Repository Naming Convention

Each repository follows a consistent naming pattern:

```
{bounded-context-name}
```

Examples:
- `portfolio-aggregation`
- `evm-integration`
- `data-models`
- `cygnus-wealth-app`

### Repository List by Domain

#### Contract Domain
- `data-models` - Shared data structures and contracts

#### Portfolio Domain
- `portfolio-aggregation` - Orchestration and aggregation logic
- `asset-valuator` - Pricing and valuation services

#### Integration Domain
- `wallet-integration-system` - Wallet connection management
- `evm-integration` - Ethereum/EVM blockchain integration
- `sol-integration` - Solana blockchain integration
- `robinhood-integration` - Traditional finance integration

#### Experience Domain
- `cygnus-wealth-app` - Main React application UI

## Topic Tags Strategy

### Required Topics

Every repository MUST include these topic tags:

**Note**: GitHub topics use hyphens, not colons (e.g., `domain-portfolio` not `domain:portfolio`)

1. **Domain Tag**: `domain-{domain-name}`
   - `domain-portfolio`
   - `domain-integration`
   - `domain-contract`
   - `domain-experience`

2. **Context Tag**: `context-{context-name}`
   - `context-aggregation`
   - `context-evm`
   - `context-solana`
   - `context-ui`

3. **Type Tag**: `type-{repository-type}`
   - `type-library`
   - `type-service`
   - `type-application`

### Optional Topics

Additional descriptive topics for discoverability:

- `blockchain`
- `defi`
- `web3`
- `react` (for UI components)
- `typescript`
- `read-only`

### Example Repository Topics

```
evm-integration:
  - domain-integration
  - context-evm
  - type-library
  - blockchain
  - web3
  - ethereum
  - typescript

portfolio-aggregation:
  - domain-portfolio
  - context-aggregation
  - type-service
  - typescript
  - defi

cygnus-wealth-app:
  - domain-experience
  - context-ui
  - type-application
  - react
  - typescript
  - web3
```

## Repository Templates

### Standard Repository Structure

Each repository should maintain this structure:

```
{repository-name}/
├── README.md                 # Domain context and documentation
├── package.json             # Package configuration
├── tsconfig.json           # TypeScript configuration
├── .github/
│   ├── workflows/          # CI/CD pipelines
│   ├── CODEOWNERS         # Code ownership
│   └── PULL_REQUEST_TEMPLATE.md
├── src/
│   ├── index.ts           # Public API exports
│   └── ...                # Domain implementation
├── tests/
│   └── ...                # Test files
└── docs/
    └── architecture.md    # Detailed architecture
```

### README Template

Each repository's README should follow this structure:

```markdown
# {Repository Name}

**Domain**: {Domain Name}  
**Bounded Context**: {Context Description}  
**Type**: {Library|Service|Application}

## Overview
Brief description of this bounded context's responsibility.

## Installation
\`\`\`bash
npm install @cygnus-wealth/{package-name}
\`\`\`

## Dependencies
- @cygnus-wealth/data-models
- Other domain dependencies

## API Documentation
Link to detailed API docs

## Architecture
Link to architectural decisions and design

## Contributing
Link to contribution guidelines
```

### Package.json Configuration

Consistent package configuration:

```json
{
  "name": "@cygnus-wealth/{repository-name}",
  "version": "1.0.0",
  "description": "{Bounded context description}",
  "keywords": [
    "domain-{domain}",
    "context-{context}",
    "type-{type}",
    "cygnus-wealth"
  ],
  "repository": {
    "type": "git",
    "url": "https://github.com/cygnus-wealth/{repository-name}"
  },
  "homepage": "https://github.com/cygnus-wealth/{repository-name}#readme",
  "bugs": {
    "url": "https://github.com/cygnus-wealth/{repository-name}/issues"
  },
  "author": "CygnusWealth",
  "license": "MIT"
}
```

## Access Control

### Team Structure

Create GitHub teams aligned with domains:

- `@cygnus-wealth/portfolio-team` - Portfolio domain maintainers
- `@cygnus-wealth/integration-team` - Integration domain maintainers
- `@cygnus-wealth/core-team` - Core application maintainers
- `@cygnus-wealth/architects` - Cross-domain architecture decisions

### CODEOWNERS File

Each repository includes a CODEOWNERS file:

```
# Global owners
* @cygnus-wealth/architects

# Domain-specific owners
/src/ @cygnus-wealth/{domain}-team
/docs/ @cygnus-wealth/architects

# Critical files
/package.json @cygnus-wealth/architects
/tsconfig.json @cygnus-wealth/architects
```

## Dependency Management

### Version Strategy

- **data-models**: Semantic versioning with strict compatibility
- **Integration domains**: Independent versioning
- **portfolio-aggregation**: Depends on integration interfaces, not implementations
- **cygnus-wealth-app**: Depends on aggregation service interface

### Dependency Rules

1. **Contract Layer** (data-models)
   - No dependencies on other domains
   - All domains can depend on this

2. **Integration Layer**
   - Depends only on data-models
   - No cross-integration dependencies

3. **Service Layer** (portfolio-aggregation, asset-valuator)
   - Depends on data-models
   - Can depend on integration interfaces

4. **Application Layer** (cygnus-wealth-app)
   - Can depend on service layer
   - Cannot directly depend on integrations

## CI/CD Configuration

### Shared Workflows

Create reusable workflows in `.github` repository:

```yaml
# .github/workflow-templates/domain-ci.yml
name: Domain CI

on:
  workflow_call:
    inputs:
      domain:
        required: true
        type: string
      context:
        required: true
        type: string

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - uses: actions/setup-node@v3
      - run: npm ci
      - run: npm test
      - run: npm run lint
```

### Repository Secrets

Organization-level secrets:
- `NPM_TOKEN` - For package publishing
- `CODECOV_TOKEN` - For coverage reporting
- `SONAR_TOKEN` - For code quality analysis

## Discovery and Navigation

### Organization README

Create a `.github` repository with profile README:

```markdown
# CygnusWealth

Decentralized portfolio aggregation platform.

## Domains

### 📊 Portfolio Domain
- [portfolio-aggregation](https://github.com/cygnus-wealth/portfolio-aggregation)
- [asset-valuator](https://github.com/cygnus-wealth/asset-valuator)

### 🔌 Integration Domain
- [wallet-integration-system](https://github.com/cygnus-wealth/wallet-integration-system)
- [evm-integration](https://github.com/cygnus-wealth/evm-integration)
- [sol-integration](https://github.com/cygnus-wealth/sol-integration)
- [robinhood-integration](https://github.com/cygnus-wealth/robinhood-integration)

### 📝 Contract Domain
- [data-models](https://github.com/cygnus-wealth/data-models)

### 💻 Experience Domain
- [cygnus-wealth-app](https://github.com/cygnus-wealth/cygnus-wealth-app)
```

### Topic-Based Discovery

Users can discover repositories by:

1. **Domain**: `org:cygnus-wealth topic:domain-integration`
2. **Type**: `org:cygnus-wealth topic:type-library`
3. **Technology**: `org:cygnus-wealth topic:blockchain`

## Migration Strategy

### Phase 1: Organization Setup
1. Create GitHub organization
2. Set up teams and permissions
3. Configure organization settings

### Phase 2: Repository Creation
1. Create repositories following naming convention
2. Apply topic tags
3. Set up CODEOWNERS

### Phase 3: Code Migration
1. Extract bounded contexts from monolith (if applicable)
2. Establish clean interfaces
3. Set up CI/CD pipelines

### Phase 4: Documentation
1. Update all READMEs
2. Document inter-domain contracts
3. Create architecture decision records

## Maintenance Guidelines

### Regular Tasks

1. **Weekly**: Review dependency updates
2. **Monthly**: Audit topic tags for consistency
3. **Quarterly**: Review team permissions
4. **Annually**: Assess domain boundaries

### Repository Health Checks

- All repositories must have:
  - README with standard sections
  - Proper topic tags
  - CODEOWNERS file
  - CI/CD pipeline
  - Security policy
  - Contribution guidelines

## Benefits of This Approach

1. **Clear Boundaries**: Each bounded context has its own repository
2. **Independent Development**: Teams can work autonomously
3. **Discoverability**: Topic tags enable easy navigation
4. **Consistent Management**: Organization-level controls
5. **Scalability**: Easy to add new domains/contexts
6. **Version Independence**: Each context can evolve separately

## Example GitHub URLs

- Organization: `https://github.com/cygnus-wealth`
- Repository: `https://github.com/cygnus-wealth/evm-integration`
- Topics Search: `https://github.com/search?q=org:cygnus-wealth+topic:domain-integration`
- Team: `https://github.com/orgs/cygnus-wealth/teams/integration-team`