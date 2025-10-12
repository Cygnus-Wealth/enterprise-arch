# Testing and Security Strategy

> **Navigation**: [Integration Domain](./README.md) > Testing & Security

## Overview

This comprehensive guide defines testing and security strategies for the Integration Domain. System Architects must ensure all integrations meet these standards to maintain data integrity, user privacy, and system reliability.

**For context**: This document is referenced from the [Integration Domain README](./README.md), which provides strategic overview and coordination guidance. Test the implementations from [Integration Patterns](./patterns.md) and [Resilience & Performance](./resilience-performance.md) using these strategies.

## Testing Strategy

### Testing Philosophy

#### Principles
1. **Test at the Right Level**: Unit test business logic, integration test external boundaries
2. **Fast Feedback**: Prioritize fast-running tests in development workflow
3. **Deterministic Results**: Tests must be repeatable and reliable
4. **Production-Like**: Test environments should mirror production conditions
5. **Continuous Validation**: Automated testing in CI/CD pipeline

### Test Pyramid Structure

```
         ┌─────────────┐
        │   E2E Tests  │  1%
       ├───────────────┤
      │ Contract Tests │  4%
     ├─────────────────┤
    │Integration Tests │  15%
   ├───────────────────┤
  │    Unit Tests      │  80%
 └─────────────────────┘
```

## Unit Testing (80% Coverage)

### What to Unit Test

#### Data Transformation Logic
```typescript
describe('EVMDataTransformer', () => {
  describe('transformBalance', () => {
    it('should convert wei to ether correctly', () => {
      const weiBalance = '1000000000000000000' // 1 ETH
      const result = transformer.transformBalance(weiBalance)

      expect(result.value).toBe('1')
      expect(result.decimals).toBe(18)
      expect(result.formatted).toBe('1.0')
    })

    it('should handle large numbers without precision loss', () => {
      const largeWei = '123456789012345678901234567890'
      const result = transformer.transformBalance(largeWei)

      // Verify BigNumber handling
      expect(result.raw).toBe(largeWei)
      expect(() => BigNumber.from(result.raw)).not.toThrow()
    })

    it('should normalize addresses to checksum format', () => {
      const lowercase = '0xabc123...'
      const result = transformer.normalizeAddress(lowercase)

      expect(result).toBe('0xAbC123...') // Checksum format
    })
  })
})
```

#### Business Rule Validation
```typescript
describe('ValidationRules', () => {
  describe('validateTransaction', () => {
    it('should reject negative values', () => {
      const tx = { value: '-100', ...validTx }
      expect(() => validator.validate(tx)).toThrow('Invalid value')
    })

    it('should enforce maximum decimal places', () => {
      const tx = { value: '1.123456789012345678901', ...validTx }
      const result = validator.validate(tx)

      expect(result.value).toBe('1.123456789012345678')
    })

    it('should validate address formats', () => {
      const invalid = ['', '0x', '0xGGG', 'not-an-address']

      invalid.forEach(addr => {
        expect(() => validator.validateAddress(addr))
          .toThrow('Invalid address format')
      })
    })
  })
})
```

#### Error Handling
```typescript
describe('ErrorHandler', () => {
  it('should classify errors correctly', () => {
    const connectionError = new Error('ECONNREFUSED')
    const result = handler.classify(connectionError)

    expect(result.type).toBe('CONNECTION')
    expect(result.retriable).toBe(true)
    expect(result.backoffStrategy).toBe('exponential')
  })

  it('should extract rate limit information', () => {
    const rateLimitError = { status: 429, headers: { 'retry-after': '60' }}
    const result = handler.handleRateLimit(rateLimitError)

    expect(result.waitTime).toBe(60000)
    expect(result.resetAt).toBeGreaterThan(Date.now())
  })
})
```

### Unit Test Patterns

#### Test Doubles Strategy
```typescript
class MockRPCProvider implements IRPCProvider {
  private responses = new Map<string, any>()

  setResponse(method: string, response: any): void {
    this.responses.set(method, response)
  }

  async call(method: string, params: any[]): Promise<any> {
    if (this.responses.has(method)) {
      return this.responses.get(method)
    }
    throw new Error(`Unexpected call: ${method}`)
  }
}

// Usage in tests
beforeEach(() => {
  mockProvider = new MockRPCProvider()
  mockProvider.setResponse('eth_getBalance', '0x1234')
  service = new BalanceService(mockProvider)
})
```

#### Parameterized Testing
```typescript
describe('Address validation', () => {
  const testCases = [
    { input: '0x' + '0'.repeat(40), valid: true },
    { input: '0x' + 'a'.repeat(40), valid: true },
    { input: '0x' + 'A'.repeat(40), valid: true },
    { input: '0x' + 'g'.repeat(40), valid: false },
    { input: 'abc123', valid: false },
    { input: '', valid: false }
  ]

  testCases.forEach(({ input, valid }) => {
    it(`should ${valid ? 'accept' : 'reject'} ${input}`, () => {
      const result = isValidAddress(input)
      expect(result).toBe(valid)
    })
  })
})
```

## Integration Testing (15% Coverage)

### Mock External Services

#### Recording and Playback Pattern
```typescript
class RecordedResponses {
  private recordings = new Map<string, Recording>()

  record(request: Request, response: Response): void {
    const key = this.generateKey(request)
    this.recordings.set(key, {
      request,
      response,
      timestamp: Date.now()
    })
  }

  playback(request: Request): Response {
    const key = this.generateKey(request)
    const recording = this.recordings.get(key)

    if (!recording) {
      throw new Error(`No recording for: ${key}`)
    }

    return recording.response
  }

  private generateKey(request: Request): string {
    return `${request.method}:${request.url}:${hash(request.params)}`
  }
}
```

#### Mock Service Implementation
```typescript
class MockBlockchainService {
  constructor(private scenario: 'normal' | 'degraded' | 'failure') {}

  async getBalance(address: string): Promise<Balance> {
    // Simulate network delay
    await this.simulateLatency()

    switch (this.scenario) {
      case 'normal':
        return this.getNormalResponse(address)
      case 'degraded':
        await this.simulateDegradation()
        return this.getDegradedResponse(address)
      case 'failure':
        throw new ConnectionError('Service unavailable')
    }
  }

  private async simulateLatency(): Promise<void> {
    const latency = 50 + Math.random() * 100 // 50-150ms
    await new Promise(resolve => setTimeout(resolve, latency))
  }
}
```

### Integration Test Scenarios

#### Happy Path Testing
```typescript
describe('Integration: EVM Balance Fetching', () => {
  it('should fetch and transform balance successfully', async () => {
    // Setup
    const mockRPC = new MockRPCServer()
    mockRPC.onCall('eth_getBalance')
      .reply('0x1234567890abcdef')

    const service = new EVMIntegrationService(mockRPC.url)

    // Execute
    const balance = await service.getBalance('0xAddress')

    // Verify
    expect(balance).toEqual({
      value: '1311768467463790320',
      formatted: '1.31',
      decimals: 18,
      symbol: 'ETH'
    })

    expect(mockRPC.calls).toHaveLength(1)
  })
})
```

#### Failure Scenario Testing
```typescript
describe('Integration: Resilience Testing', () => {
  it('should fallback to secondary provider on primary failure', async () => {
    const primary = new MockRPCServer()
    primary.failWithError('ECONNREFUSED')

    const secondary = new MockRPCServer()
    secondary.onCall('eth_getBalance').reply('0x100')

    const service = new EVMIntegrationService({
      providers: [primary.url, secondary.url]
    })

    const balance = await service.getBalance('0xAddress')

    expect(balance).toBeDefined()
    expect(primary.calls).toHaveLength(1)
    expect(secondary.calls).toHaveLength(1)
  })

  it('should return cached data when all providers fail', async () => {
    // Prime cache
    await service.getBalance('0xAddress')

    // All providers fail
    mockProviders.forEach(p => p.fail())

    const balance = await service.getBalance('0xAddress')

    expect(balance.fromCache).toBe(true)
    expect(balance.cacheAge).toBeLessThan(60000)
  })
})
```

## Contract Testing (4% Coverage)

### Data Contract Validation

```typescript
describe('Contract: Data Model Compliance', () => {
  const validator = new DataModelValidator()

  it('should validate Asset contract', () => {
    const asset = {
      id: 'eth-mainnet',
      symbol: 'ETH',
      name: 'Ethereum',
      decimals: 18,
      chainId: 1,
      address: null, // Native token
      logoUrl: 'https://...',
      priceUSD: '2000.50'
    }

    expect(validator.validateAsset(asset)).toBe(true)
  })

  it('should reject invalid Transaction contract', () => {
    const invalid = {
      hash: '0xabc', // Too short
      value: 'not-a-number',
      timestamp: 'not-a-timestamp'
    }

    const errors = validator.validateTransaction(invalid)

    expect(errors).toContain('Invalid hash format')
    expect(errors).toContain('Invalid value type')
    expect(errors).toContain('Invalid timestamp')
  })
})
```

### API Contract Testing

```typescript
describe('Contract: Integration API', () => {
  it('should return expected response schema', async () => {
    const response = await service.getPortfolio('0xAddress')

    // Validate schema
    expect(response).toMatchSchema({
      type: 'object',
      required: ['assets', 'totalValueUSD', 'lastUpdated'],
      properties: {
        assets: {
          type: 'array',
          items: { $ref: '#/definitions/Asset' }
        },
        totalValueUSD: { type: 'string' },
        lastUpdated: { type: 'number' }
      }
    })
  })
})
```

## End-to-End Testing (1% Coverage)

### Critical Path Testing

```typescript
describe('E2E: Portfolio Loading', () => {
  it('should load complete portfolio for connected wallet', async () => {
    // Connect wallet
    await walletService.connect('metamask')

    // Wait for address
    const address = await walletService.getAddress()

    // Trigger portfolio load
    await portfolioService.loadPortfolio(address)

    // Verify all integrations worked
    const portfolio = await portfolioService.getPortfolio()

    expect(portfolio.assets).toHaveLength(greaterThan(0))
    expect(portfolio.chains).toContain('ethereum')
    expect(portfolio.totalValue).toBeGreaterThan(0)
  })
})
```

## Security Strategy

### Security Principles

1. **Zero Trust**: Never trust external data without validation
2. **Defense in Depth**: Multiple layers of security controls
3. **Least Privilege**: Minimal permissions for all operations
4. **Fail Secure**: Default to secure state on failures
5. **Privacy by Design**: No unnecessary data collection

### Read-Only Guarantee Implementation

#### Code Analysis Rules
```typescript
// ESLint rules for read-only enforcement
module.exports = {
  rules: {
    'no-private-keys': {
      create(context) {
        return {
          Literal(node) {
            if (node.value && typeof node.value === 'string') {
              // Check for private key patterns
              const patterns = [
                /^0x[a-fA-F0-9]{64}$/,  // Private key
                /^[a-zA-Z0-9]{64}$/,    // Seed phrase word
                /private/i,             // Variable naming
                /secret/i,              // Secret naming
              ]

              patterns.forEach(pattern => {
                if (pattern.test(node.value)) {
                  context.report({
                    node,
                    message: 'Potential private key detected'
                  })
                }
              })
            }
          }
        }
      }
    }
  }
}
```

#### Runtime Protection
```typescript
class ReadOnlyGuard {
  constructor() {
    this.interceptDangerousMethods()
  }

  private interceptDangerousMethods(): void {
    // Block transaction signing
    const dangerousMethods = [
      'eth_sendTransaction',
      'eth_sendRawTransaction',
      'eth_sign',
      'personal_sign',
      'eth_signTypedData'
    ]

    dangerousMethods.forEach(method => {
      this.blockMethod(method)
    })
  }

  private blockMethod(method: string): void {
    Object.defineProperty(window.ethereum, method, {
      value: () => {
        throw new SecurityError(`Method ${method} is blocked in read-only mode`)
      },
      writable: false,
      configurable: false
    })
  }
}
```

### Input Validation and Sanitization

#### Address Validation
```typescript
class AddressValidator {
  validate(address: string, chain: string): boolean {
    switch (chain) {
      case 'ethereum':
        return this.validateEVMAddress(address)
      case 'solana':
        return this.validateSolanaAddress(address)
      default:
        throw new Error(`Unknown chain: ${chain}`)
    }
  }

  private validateEVMAddress(address: string): boolean {
    // Check format
    if (!/^0x[a-fA-F0-9]{40}$/.test(address)) {
      return false
    }

    // Verify checksum if present
    if (address !== address.toLowerCase() &&
        address !== address.toUpperCase()) {
      return this.verifyChecksum(address)
    }

    return true
  }

  private validateSolanaAddress(address: string): boolean {
    // Base58 validation
    try {
      const decoded = bs58.decode(address)
      return decoded.length === 32
    } catch {
      return false
    }
  }
}
```

#### Data Sanitization
```typescript
class DataSanitizer {
  sanitize(data: any): any {
    if (typeof data === 'string') {
      return this.sanitizeString(data)
    }

    if (Array.isArray(data)) {
      return data.map(item => this.sanitize(item))
    }

    if (typeof data === 'object' && data !== null) {
      const sanitized = {}
      for (const [key, value] of Object.entries(data)) {
        // Skip potentially dangerous keys
        if (this.isDangerousKey(key)) {
          continue
        }
        sanitized[key] = this.sanitize(value)
      }
      return sanitized
    }

    return data
  }

  private sanitizeString(str: string): string {
    return str
      .replace(/<script\b[^<]*(?:(?!<\/script>)<[^<]*)*<\/script>/gi, '')
      .replace(/javascript:/gi, '')
      .replace(/on\w+\s*=/gi, '')
      .trim()
  }

  private isDangerousKey(key: string): boolean {
    const dangerous = [
      'privateKey',
      'secretKey',
      'password',
      'mnemonic',
      'seed'
    ]
    return dangerous.some(d => key.toLowerCase().includes(d))
  }
}
```

### API Security

#### Request Authentication
```typescript
class SecureAPIClient {
  private readonly apiKey: string
  private readonly rateLimiter: RateLimiter

  async request(endpoint: string, params: any): Promise<any> {
    // Rate limiting
    await this.rateLimiter.acquire()

    // Build secure request
    const request = {
      url: this.buildSecureUrl(endpoint),
      headers: this.getSecureHeaders(),
      params: this.sanitizeParams(params)
    }

    // Sign request if required
    if (this.requiresSigning(endpoint)) {
      request.headers['X-Signature'] = this.signRequest(request)
    }

    try {
      const response = await fetch(request.url, {
        headers: request.headers,
        // Never include credentials
        credentials: 'omit'
      })

      return this.validateResponse(response)
    } catch (error) {
      this.handleSecurityError(error)
    }
  }

  private getSecureHeaders(): Headers {
    return {
      'X-API-Key': this.apiKey,
      'X-Request-ID': crypto.randomUUID(),
      'X-Timestamp': Date.now().toString(),
      // Prevent CSRF
      'X-Requested-With': 'XMLHttpRequest'
    }
  }
}
```

### Secure Storage

#### API Credential Management
```typescript
class SecureCredentialStore {
  private cipher: Cipher

  async storeAPIKey(provider: string, key: string): Promise<void> {
    // Encrypt before storage
    const encrypted = await this.cipher.encrypt(key)

    // Store in secure location (not localStorage)
    await this.secureStore.set(`api:${provider}`, encrypted)

    // Set expiration
    await this.setExpiration(provider, Date.now() + 86400000) // 24h
  }

  async getAPIKey(provider: string): Promise<string | null> {
    // Check expiration
    if (await this.isExpired(provider)) {
      await this.secureStore.delete(`api:${provider}`)
      return null
    }

    // Retrieve and decrypt
    const encrypted = await this.secureStore.get(`api:${provider}`)

    if (!encrypted) {
      return null
    }

    return await this.cipher.decrypt(encrypted)
  }

  // Never store in code or config files
  private validateKeyStorage(key: string): void {
    if (key.includes('hardcoded')) {
      throw new SecurityError('Never hardcode API keys')
    }
  }
}
```

### Privacy Protection

#### Data Minimization
```typescript
class PrivacyManager {
  // Only collect necessary data
  filterPersonalData(data: any): any {
    const filtered = { ...data }

    // Remove unnecessary personal information
    delete filtered.email
    delete filtered.phone
    delete filtered.fullName
    delete filtered.ipAddress

    // Truncate addresses for privacy
    if (filtered.address) {
      filtered.address = this.truncateAddress(filtered.address)
    }

    return filtered
  }

  private truncateAddress(address: string): string {
    // Show only first 6 and last 4 characters
    return `${address.slice(0, 6)}...${address.slice(-4)}`
  }
}
```

#### Analytics Privacy
```typescript
class PrivateAnalytics {
  track(event: string, properties: any): void {
    // No external tracking services
    // Local analytics only

    const sanitized = {
      event,
      timestamp: Date.now(),
      // Hash identifiers
      userId: this.hash(properties.userId),
      // Remove sensitive data
      properties: this.sanitizeProperties(properties)
    }

    // Store locally only
    this.localStore.append('analytics', sanitized)
  }

  private sanitizeProperties(props: any): any {
    const safe = {}

    for (const [key, value] of Object.entries(props)) {
      // Skip sensitive fields
      if (this.isSensitive(key)) continue

      // Hash addresses
      if (key.includes('address')) {
        safe[key] = this.hash(value)
      } else {
        safe[key] = value
      }
    }

    return safe
  }
}
```

### Security Audit Checklist

#### Code Review Checklist
- [ ] No private key handling code
- [ ] All inputs validated and sanitized
- [ ] No hardcoded credentials
- [ ] Proper error handling without info leakage
- [ ] Rate limiting implemented
- [ ] CORS properly configured
- [ ] Dependencies up to date
- [ ] No eval() or Function() usage
- [ ] Content Security Policy implemented

#### Runtime Security Checks
```typescript
class SecurityAuditor {
  async auditIntegration(integration: Integration): Promise<AuditReport> {
    const checks = [
      this.checkReadOnly(integration),
      this.checkInputValidation(integration),
      this.checkErrorHandling(integration),
      this.checkRateLimiting(integration),
      this.checkDataSanitization(integration),
      this.checkPrivacy(integration)
    ]

    const results = await Promise.all(checks)

    return {
      passed: results.every(r => r.passed),
      findings: results.flatMap(r => r.findings),
      score: this.calculateScore(results)
    }
  }
}
```

## Testing Best Practices

### Test Data Management

```typescript
class TestDataFactory {
  // Deterministic test data generation
  createAddress(seed: number): string {
    const bytes = crypto.hash(`address:${seed}`)
    return `0x${bytes.slice(0, 40)}`
  }

  createTransaction(overrides?: Partial<Transaction>): Transaction {
    return {
      hash: `0x${'a'.repeat(64)}`,
      from: this.createAddress(1),
      to: this.createAddress(2),
      value: '1000000000000000000',
      timestamp: Date.now(),
      ...overrides
    }
  }
}
```

### Test Environment Setup

```typescript
// test-setup.ts
export async function setupTestEnvironment(): Promise<TestEnv> {
  // Create isolated IndexedDB for tests
  const db = await createTestDatabase()

  // Setup mock providers
  const providers = createMockProviders()

  // Initialize services with test config
  const services = await initializeServices({
    database: db,
    providers,
    cache: new MemoryCache(),
    logger: new TestLogger()
  })

  return {
    db,
    providers,
    services,
    cleanup: async () => {
      await db.clear()
      providers.forEach(p => p.stop())
    }
  }
}
```

## Summary

This testing and security strategy ensures:

1. **Comprehensive Testing**: Full coverage across all test levels
2. **Security by Design**: Read-only guarantee enforced at multiple levels
3. **Privacy Protection**: User data never exposed or tracked
4. **Reliable Integration**: Deterministic and maintainable tests
5. **Continuous Validation**: Automated security and quality checks

System Architects must implement these strategies to maintain the integrity, security, and reliability of the Integration Domain.

---

## Related Documentation

- **[← Integration Domain README](./README.md)** - Domain overview and strategic guidance
- **[Integration Patterns Guide](./patterns.md)** - Patterns to test and secure
- **[Resilience & Performance Guide](./resilience-performance.md)** - Test resilience and performance characteristics
- **[Bounded Contexts](./bounded-contexts/)** - Apply testing strategies to each context