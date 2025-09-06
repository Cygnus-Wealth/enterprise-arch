# Wallet Integration System Domain

## Domain Overview

The Wallet Integration System domain manages multi-chain wallet connections across the CygnusWealth ecosystem. It provides a unified interface for connecting, managing, and switching between wallets on Ethereum/EVM, Solana, and SUI blockchains, focusing exclusively on wallet connection management without handling any transaction signing or private keys.

## Core Responsibilities

### Primary Functions
1. **Wallet Discovery**: Detect available wallet providers
2. **Connection Management**: Connect and disconnect wallets
3. **Multi-Account Support**: Handle multiple accounts per wallet
4. **Chain Switching**: Coordinate network changes across wallets
5. **Session Persistence**: Maintain wallet connections across sessions
6. **Event Handling**: React to wallet state changes

### Domain Boundaries
- **Owns**: Wallet connections, provider management, session state
- **Does NOT Own**: Transaction signing, balance fetching, private keys

## Architecture

### Core Manager
```typescript
class WalletManager {
  // Connection management
  async connect(walletType: WalletType, chain: Chain): Promise<WalletConnection>
  async disconnect(walletId: string): Promise<void>
  async disconnectAll(): Promise<void>
  
  // Wallet operations
  async switchAccount(walletId: string, accountIndex: number): Promise<void>
  async switchChain(walletId: string, chainId: number): Promise<void>
  async refreshConnection(walletId: string): Promise<void>
  
  // State queries
  getConnectedWallets(): WalletConnection[]
  getWallet(walletId: string): WalletConnection | null
  isWalletConnected(walletType: WalletType): boolean
  getSupportedWallets(): WalletInfo[]
  
  // Events
  onConnect(callback: (wallet: WalletConnection) => void): Unsubscribe
  onDisconnect(callback: (walletId: string) => void): Unsubscribe
  onAccountChange(callback: (wallet: WalletConnection) => void): Unsubscribe
  onChainChange(callback: (wallet: WalletConnection) => void): Unsubscribe
}
```

## Wallet Providers

### EVM Wallets
```typescript
interface EvmWalletProvider {
  // MetaMask
  detectMetaMask(): boolean
  connectMetaMask(): Promise<EvmWalletConnection>
  
  // WalletConnect
  initWalletConnect(projectId: string): void
  connectWalletConnect(): Promise<EvmWalletConnection>
  
  // Coinbase Wallet
  detectCoinbaseWallet(): boolean
  connectCoinbaseWallet(): Promise<EvmWalletConnection>
  
  // Rainbow
  detectRainbow(): boolean
  connectRainbow(): Promise<EvmWalletConnection>
}

interface EvmWalletConnection {
  provider: EthereumProvider
  accounts: string[]
  chainId: number
  walletType: 'metamask' | 'walletconnect' | 'coinbase' | 'rainbow'
}
```

### Solana Wallets
```typescript
interface SolanaWalletProvider {
  // Phantom
  detectPhantom(): boolean
  connectPhantom(): Promise<SolanaWalletConnection>
  
  // Solflare
  detectSolflare(): boolean
  connectSolflare(): Promise<SolanaWalletConnection>
  
  // Backpack
  detectBackpack(): boolean
  connectBackpack(): Promise<SolanaWalletConnection>
  
  // Slush
  detectSlush(): boolean
  connectSlush(): Promise<SolanaWalletConnection>
}

interface SolanaWalletConnection {
  publicKey: PublicKey
  walletType: 'phantom' | 'solflare' | 'backpack' | 'slush'
  isConnected: boolean
}
```

### SUI Wallets
```typescript
interface SuiWalletProvider {
  // Sui Wallet
  detectSuiWallet(): boolean
  connectSuiWallet(): Promise<SuiWalletConnection>
  
  // Suiet
  detectSuiet(): boolean
  connectSuiet(): Promise<SuiWalletConnection>
  
  // Ethos
  detectEthos(): boolean
  connectEthos(): Promise<SuiWalletConnection>
}

interface SuiWalletConnection {
  address: string
  walletType: 'sui' | 'suiet' | 'ethos'
  features: SuiFeatures
}
```

## Data Models

### Wallet Connection
```typescript
interface WalletConnection {
  id: string                    // Unique identifier
  type: WalletType              // Provider type
  chain: Chain                  // Current chain
  accounts: Account[]           // Available accounts
  activeAccount: Account        // Currently selected account
  status: ConnectionStatus      // Connection state
  connectedAt: Date            // Connection timestamp
  metadata: WalletMetadata     // Additional info
}

interface Account {
  address: string
  label?: string
  isActive: boolean
}

enum ConnectionStatus {
  CONNECTED = 'connected',
  CONNECTING = 'connecting',
  DISCONNECTED = 'disconnected',
  ERROR = 'error'
}

interface WalletMetadata {
  name: string
  icon: string
  version?: string
  features: string[]
}
```

### Wallet Types
```typescript
enum WalletType {
  // EVM Wallets
  METAMASK = 'metamask',
  WALLETCONNECT = 'walletconnect',
  COINBASE = 'coinbase',
  RAINBOW = 'rainbow',
  
  // Solana Wallets
  PHANTOM = 'phantom',
  SOLFLARE = 'solflare',
  BACKPACK = 'backpack',
  SLUSH = 'slush',
  
  // SUI Wallets
  SUI_WALLET = 'sui_wallet',
  SUIET = 'suiet',
  ETHOS = 'ethos'
}

interface WalletInfo {
  type: WalletType
  name: string
  icon: string
  chains: Chain[]
  downloadUrl?: string
  deepLink?: string
  isInstalled: boolean
}
```

## Connection Flow

### Connection Sequence
```typescript
class ConnectionFlow {
  async initiateConnection(walletType: WalletType): Promise<WalletConnection> {
    // 1. Check if wallet is available
    const isAvailable = await this.checkAvailability(walletType)
    if (!isAvailable) {
      throw new WalletNotFoundError(walletType)
    }
    
    // 2. Request connection
    const provider = await this.getProvider(walletType)
    const accounts = await provider.requestAccounts()
    
    // 3. Get chain information
    const chainId = await provider.getChainId()
    
    // 4. Create connection object
    const connection = this.createConnection({
      type: walletType,
      accounts,
      chainId
    })
    
    // 5. Register event listeners
    this.registerListeners(connection)
    
    // 6. Persist connection
    await this.persistConnection(connection)
    
    return connection
  }
}
```

### Event Handling
```typescript
class WalletEventHandler {
  private listeners: Map<string, EventListener[]>
  
  registerProvider(provider: any, walletId: string): void {
    // Account change events
    provider.on('accountsChanged', (accounts: string[]) => {
      this.handleAccountsChanged(walletId, accounts)
    })
    
    // Chain change events
    provider.on('chainChanged', (chainId: string) => {
      this.handleChainChanged(walletId, chainId)
    })
    
    // Disconnect events
    provider.on('disconnect', () => {
      this.handleDisconnect(walletId)
    })
  }
  
  private handleAccountsChanged(walletId: string, accounts: string[]): void {
    if (accounts.length === 0) {
      // Wallet disconnected
      this.disconnectWallet(walletId)
    } else {
      // Update accounts
      this.updateAccounts(walletId, accounts)
      this.emit('accountsChanged', { walletId, accounts })
    }
  }
}
```

## Multi-Wallet Management

### Wallet Registry
```typescript
class WalletRegistry {
  private connections: Map<string, WalletConnection>
  private providers: Map<string, any>
  
  register(connection: WalletConnection, provider: any): void {
    this.connections.set(connection.id, connection)
    this.providers.set(connection.id, provider)
  }
  
  unregister(walletId: string): void {
    this.connections.delete(walletId)
    this.providers.delete(walletId)
  }
  
  getByChain(chain: Chain): WalletConnection[] {
    return Array.from(this.connections.values())
      .filter(conn => conn.chain === chain)
  }
  
  getByType(type: WalletType): WalletConnection[] {
    return Array.from(this.connections.values())
      .filter(conn => conn.type === type)
  }
}
```

### Session Management
```typescript
class SessionManager {
  private readonly STORAGE_KEY = 'wallet_sessions'
  
  async saveSession(connection: WalletConnection): Promise<void> {
    const sessions = await this.loadSessions()
    sessions[connection.id] = {
      type: connection.type,
      chain: connection.chain,
      accounts: connection.accounts,
      connectedAt: connection.connectedAt
    }
    await this.persistSessions(sessions)
  }
  
  async restoreSessions(): Promise<WalletConnection[]> {
    const sessions = await this.loadSessions()
    const restored: WalletConnection[] = []
    
    for (const [id, session] of Object.entries(sessions)) {
      try {
        const connection = await this.reconnect(session)
        restored.push(connection)
      } catch (error) {
        // Session expired or wallet unavailable
        delete sessions[id]
      }
    }
    
    await this.persistSessions(sessions)
    return restored
  }
}
```

## Chain Management

### Network Configuration
```typescript
interface NetworkConfig {
  chainId: number
  name: string
  rpcUrl?: string
  nativeCurrency: {
    name: string
    symbol: string
    decimals: number
  }
  blockExplorer?: string
}

class ChainManager {
  private networks: Map<number, NetworkConfig>
  
  async switchChain(
    walletId: string,
    targetChainId: number
  ): Promise<void> {
    const wallet = this.getWallet(walletId)
    const provider = this.getProvider(walletId)
    
    try {
      // Try to switch to existing chain
      await provider.request({
        method: 'wallet_switchEthereumChain',
        params: [{ chainId: `0x${targetChainId.toString(16)}` }]
      })
    } catch (error) {
      // Chain doesn't exist, add it
      if (error.code === 4902) {
        const network = this.networks.get(targetChainId)
        await this.addChain(provider, network)
      } else {
        throw error
      }
    }
  }
  
  private async addChain(
    provider: any,
    network: NetworkConfig
  ): Promise<void> {
    await provider.request({
      method: 'wallet_addEthereumChain',
      params: [{
        chainId: `0x${network.chainId.toString(16)}`,
        chainName: network.name,
        rpcUrls: [network.rpcUrl],
        nativeCurrency: network.nativeCurrency,
        blockExplorerUrls: [network.blockExplorer]
      }]
    })
  }
}
```

## Error Handling

### Error Types
```typescript
enum WalletErrorCode {
  WALLET_NOT_FOUND = 'WALLET_NOT_FOUND',
  CONNECTION_REJECTED = 'CONNECTION_REJECTED',
  ALREADY_CONNECTED = 'ALREADY_CONNECTED',
  CHAIN_NOT_SUPPORTED = 'CHAIN_NOT_SUPPORTED',
  ACCOUNT_ACCESS_DENIED = 'ACCOUNT_ACCESS_DENIED',
  PROVIDER_ERROR = 'PROVIDER_ERROR',
  SESSION_EXPIRED = 'SESSION_EXPIRED'
}

class WalletError extends Error {
  constructor(
    message: string,
    readonly code: WalletErrorCode,
    readonly walletType?: WalletType,
    readonly details?: any
  ) {
    super(message)
  }
}
```

### Error Recovery
```typescript
class ErrorRecovery {
  async handleConnectionError(
    error: WalletError
  ): Promise<RecoveryAction> {
    switch (error.code) {
      case WalletErrorCode.WALLET_NOT_FOUND:
        return {
          action: 'SHOW_INSTALL_PROMPT',
          walletType: error.walletType
        }
      
      case WalletErrorCode.CONNECTION_REJECTED:
        return {
          action: 'RETRY_CONNECTION',
          delay: 1000
        }
      
      case WalletErrorCode.SESSION_EXPIRED:
        return {
          action: 'RECONNECT',
          clearSession: true
        }
      
      default:
        return {
          action: 'SHOW_ERROR',
          message: error.message
        }
    }
  }
}
```

## Testing Strategy

### Unit Tests
```typescript
describe('WalletManager', () => {
  test('connects to MetaMask successfully', async () => {
    const mockProvider = createMockMetaMaskProvider()
    const manager = new WalletManager()
    
    const connection = await manager.connect(
      WalletType.METAMASK,
      Chain.ETHEREUM
    )
    
    expect(connection.type).toBe(WalletType.METAMASK)
    expect(connection.status).toBe(ConnectionStatus.CONNECTED)
  })
  
  test('handles multiple wallet connections', async () => {
    const manager = new WalletManager()
    
    await manager.connect(WalletType.METAMASK, Chain.ETHEREUM)
    await manager.connect(WalletType.PHANTOM, Chain.SOLANA)
    
    const wallets = manager.getConnectedWallets()
    expect(wallets).toHaveLength(2)
  })
})
```

### Integration Tests
- Multi-wallet connection scenarios
- Chain switching workflows
- Session persistence and recovery
- Event handling across wallets

## Configuration

### System Configuration
```typescript
interface WalletSystemConfig {
  // Connection settings
  autoConnect: boolean           // Auto-reconnect on page load
  connectionTimeout: number       // Connection attempt timeout
  maxReconnectAttempts: number  // Max reconnection attempts
  
  // Session settings
  persistSessions: boolean       // Save wallet sessions
  sessionExpiry: number          // Session expiry time (ms)
  
  // WalletConnect settings
  walletConnectProjectId?: string
  walletConnectRelayUrl?: string
  
  // Feature flags
  enableMultiWallet: boolean     // Allow multiple wallets
  enableChainSwitching: boolean  // Allow chain switching
  showUnsupportedWallets: boolean // Show non-installed wallets
}
```

## Usage Example

```typescript
// Initialize wallet manager
const walletManager = new WalletManager({
  autoConnect: true,
  persistSessions: true,
  walletConnectProjectId: 'your-project-id'
})

// Connect to MetaMask
const metamask = await walletManager.connect(
  WalletType.METAMASK,
  Chain.ETHEREUM
)

// Get connected address
const address = metamask.activeAccount.address

// Switch chain
await walletManager.switchChain(metamask.id, 137) // Switch to Polygon

// Listen for account changes
walletManager.onAccountChange((wallet) => {
  console.log('Account changed:', wallet.activeAccount.address)
})

// Connect to Phantom (Solana)
const phantom = await walletManager.connect(
  WalletType.PHANTOM,
  Chain.SOLANA
)

// Get all connected wallets
const wallets = walletManager.getConnectedWallets()

// Disconnect all
await walletManager.disconnectAll()
```

## Future Enhancements

### Planned Features
1. **Hardware Wallet Support**: Ledger and Trezor integration
2. **Mobile Wallet Support**: Deep linking for mobile wallets
3. **Multi-Signature Support**: Gnosis Safe and similar
4. **Wallet Analytics**: Connection metrics and usage patterns
5. **Custom Wallet Adapters**: Plugin system for new wallets
6. **QR Code Connections**: Enhanced mobile connection flow