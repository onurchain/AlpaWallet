# 🔧 Technical Deep Dive - Alpa Wallet

## Architecture Overview

Alpa Wallet is built using a modern, scalable architecture that prioritizes security, performance, and user experience. The application follows a clean separation of concerns with distinct frontend, backend, and blockchain integration layers.

## Frontend Architecture

### Component Structure
```typescript
// Example: Core Wallet Hook Implementation
export function useWallet() {
  const [walletState, setWalletState] = useState<WalletState>({
    wallets: [],
    activeWallet: null,
    tokens: [],
    customTokens: [],
    isLoading: false,
    totalBalanceUsd: 0
  });

  // Real-time balance synchronization
  const syncBalances = useCallback(async () => {
    const web3 = new Web3(BSC_RPC_URL);
    const balances = await Promise.all(
      tokens.map(token => getTokenBalance(web3, token, activeWallet.address))
    );
    
    setWalletState(prev => ({
      ...prev,
      tokens: updateTokenBalances(prev.tokens, balances)
    }));
  }, [tokens, activeWallet]);

  return { walletState, syncBalances, createWallet, importWallet };
}
```

### State Management Pattern
- **React Context**: Global wallet state management
- **TanStack Query**: Server state synchronization
- **Local Storage**: Encrypted wallet persistence
- **WebSocket**: Real-time updates

### UI Component System
```typescript
// Example: Token Card Component
interface TokenCardProps {
  token: Token;
  onClick?: () => void;
  onSend?: (symbol: string) => void;
  onReceive?: (symbol: string) => void;
}

export function TokenCard({ token, onClick, onSend, onReceive }: TokenCardProps) {
  return (
    <Card className="p-4 hover:bg-accent/50 transition-colors">
      <div className="flex items-center justify-between">
        <div className="flex items-center space-x-3">
          <Avatar className="h-10 w-10">
            <AvatarImage src={token.logoUrl} alt={token.symbol} />
            <AvatarFallback>{token.symbol.slice(0, 2)}</AvatarFallback>
          </Avatar>
          <div>
            <p className="font-medium">{token.name}</p>
            <p className="text-sm text-muted-foreground">{token.symbol}</p>
          </div>
        </div>
        <div className="text-right">
          <p className="font-semibold">${token.usdValue.toFixed(2)}</p>
          <p className="text-sm text-muted-foreground">
            {parseFloat(token.balance).toLocaleString()} {token.symbol}
          </p>
        </div>
      </div>
    </Card>
  );
}
```

## Backend Implementation

### API Layer
```typescript
// Example: Token Purchase API
app.post('/api/purchased-tokens', async (req, res) => {
  try {
    const validatedData = purchasedTokenSchema.parse(req.body);
    
    // Verify wallet ownership
    const wallet = await verifyWalletOwnership(validatedData.walletAddress);
    if (!wallet) {
      return res.status(403).json({ error: 'Wallet verification failed' });
    }

    // Store token purchase record
    const [newPurchase] = await db.insert(purchasedTokens)
      .values({
        ...validatedData,
        createdAt: new Date(),
        status: 'confirmed'
      })
      .returning();

    res.status(201).json({ success: true, data: newPurchase });
  } catch (error) {
    handleApiError(error, res);
  }
});
```

### Database Schema
```typescript
// Drizzle ORM Schema Example
export const wallets = pgTable('wallets', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  address: text('address').notNull().unique(),
  encryptedKey: text('encrypted_key').notNull(),
  hasBackedUp: boolean('has_backed_up').default(false),
  userId: integer('user_id').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull()
});

export const purchasedTokens = pgTable('purchased_tokens', {
  id: serial('id').primaryKey(),
  walletAddress: text('wallet_address').notNull(),
  symbol: text('symbol').notNull(),
  amount: text('amount').notNull(),
  usdValue: decimal('usd_value', { precision: 10, scale: 2 }),
  transactionHash: text('transaction_hash'),
  status: text('status').notNull().default('pending'),
  createdAt: timestamp('created_at').defaultNow().notNull()
});
```

## Blockchain Integration

### Web3 Implementation
```typescript
// Example: Token Balance Fetching
async function getTokenBalance(
  web3: Web3, 
  tokenAddress: string, 
  walletAddress: string
): Promise<string> {
  const contract = new web3.eth.Contract(ERC20_ABI, tokenAddress);
  
  try {
    const balance = await contract.methods.balanceOf(walletAddress).call();
    const decimals = await contract.methods.decimals().call();
    
    return web3.utils.fromWei(
      balance.toString(), 
      getWeiUnit(decimals)
    );
  } catch (error) {
    console.error(`Error fetching balance for ${tokenAddress}:`, error);
    return '0';
  }
}

// Transaction Sending
async function sendToken(
  web3: Web3,
  fromAddress: string,
  toAddress: string,
  amount: string,
  tokenAddress: string,
  privateKey: string
): Promise<TransactionReceipt> {
  const contract = new web3.eth.Contract(ERC20_ABI, tokenAddress);
  const decimals = await contract.methods.decimals().call();
  const amountWei = web3.utils.toWei(amount, getWeiUnit(decimals));

  const transaction = contract.methods.transfer(toAddress, amountWei);
  const gas = await transaction.estimateGas({ from: fromAddress });
  const gasPrice = await web3.eth.getGasPrice();

  const signedTx = await web3.eth.accounts.signTransaction({
    to: tokenAddress,
    data: transaction.encodeABI(),
    gas: gas.toString(),
    gasPrice: gasPrice.toString(),
    nonce: await web3.eth.getTransactionCount(fromAddress)
  }, privateKey);

  return await web3.eth.sendSignedTransaction(signedTx.rawTransaction!);
}
```

### Wallet Generation
```typescript
// Secure Wallet Creation
import { generateMnemonic, mnemonicToSeedSync } from 'bip39';
import { fromSeed } from 'hdkey';
import { encrypt, decrypt } from './crypto';

export function createNewWallet(password: string): WalletData {
  // Generate BIP39 mnemonic
  const mnemonic = generateMnemonic(128); // 12 words
  const seed = mnemonicToSeedSync(mnemonic);
  
  // Derive HD wallet
  const hdkey = fromSeed(seed);
  const wallet = hdkey.derivePath("m/44'/60'/0'/0/0"); // Ethereum derivation path
  
  const privateKey = wallet.privateKey.toString('hex');
  const address = `0x${wallet.publicKey.slice(-40)}`;
  
  // Encrypt sensitive data
  const encryptedMnemonic = encrypt(mnemonic, password);
  const encryptedPrivateKey = encrypt(privateKey, password);
  
  return {
    address,
    encryptedMnemonic,
    encryptedPrivateKey,
    derivationPath: "m/44'/60'/0'/0/0"
  };
}
```

## Security Implementation

### Encryption System
```typescript
// AES-256 Encryption Utilities
import CryptoJS from 'crypto-js';

export function encrypt(text: string, password: string): string {
  const salt = CryptoJS.lib.WordArray.random(128/8);
  const key = CryptoJS.PBKDF2(password, salt, {
    keySize: 256/32,
    iterations: 10000
  });
  
  const encrypted = CryptoJS.AES.encrypt(text, key, {
    iv: CryptoJS.lib.WordArray.random(128/8),
    padding: CryptoJS.pad.Pkcs7,
    mode: CryptoJS.mode.CBC
  });
  
  return salt.toString() + encrypted.toString();
}

export function decrypt(encryptedText: string, password: string): string {
  const salt = CryptoJS.enc.Hex.parse(encryptedText.substr(0, 32));
  const encrypted = encryptedText.substring(32);
  
  const key = CryptoJS.PBKDF2(password, salt, {
    keySize: 256/32,
    iterations: 10000
  });
  
  const decrypted = CryptoJS.AES.decrypt(encrypted, key, {
    padding: CryptoJS.pad.Pkcs7,
    mode: CryptoJS.mode.CBC
  });
  
  return decrypted.toString(CryptoJS.enc.Utf8);
}
```

### Input Validation
```typescript
// Zod Schema Validation
import { z } from 'zod';

export const walletSchema = z.object({
  name: z.string().min(1, 'Wallet name is required').max(50),
  address: z.string().regex(/^0x[a-fA-F0-9]{40}$/, 'Invalid Ethereum address'),
  encryptedKey: z.string().min(1, 'Encrypted key is required')
});

export const transactionSchema = z.object({
  from: z.string().regex(/^0x[a-fA-F0-9]{40}$/),
  to: z.string().regex(/^0x[a-fA-F0-9]{40}$/),
  amount: z.string().refine(val => !isNaN(parseFloat(val)) && parseFloat(val) > 0),
  tokenAddress: z.string().regex(/^0x[a-fA-F0-9]{40}$/).optional()
});
```

## Performance Optimizations

### Code Splitting
```typescript
// Lazy Loading Implementation
import { lazy, Suspense } from 'react';

const WalletScreen = lazy(() => import('./components/wallet/WalletScreen'));
const TokenPurchaseScreen = lazy(() => import('./components/wallet/TokenPurchaseScreen'));
const MarketsScreen = lazy(() => import('./components/markets/MarketsScreen'));

export function App() {
  return (
    <Router>
      <Suspense fallback={<LoadingSpinner />}>
        <Route path="/" component={WalletScreen} />
        <Route path="/purchase" component={TokenPurchaseScreen} />
        <Route path="/markets" component={MarketsScreen} />
      </Suspense>
    </Router>
  );
}
```

### Caching Strategy
```typescript
// TanStack Query Configuration
export const queryClient = new QueryClient({
  defaultOptions: {
    queries: {
      staleTime: 5 * 60 * 1000, // 5 minutes
      cacheTime: 10 * 60 * 1000, // 10 minutes
      retry: 3,
      refetchOnWindowFocus: false
    },
    mutations: {
      onError: (error) => {
        toast.error(`Transaction failed: ${error.message}`);
      }
    }
  }
});

// Price Data Caching
export function useTokenPrices() {
  return useQuery({
    queryKey: ['tokenPrices'],
    queryFn: fetchTokenPrices,
    staleTime: 2 * 60 * 1000, // 2 minutes for price data
    refetchInterval: 2 * 60 * 1000
  });
}
```

## Testing Strategy

### Unit Tests Example
```typescript
// Wallet Creation Tests
describe('Wallet Creation', () => {
  test('should create valid wallet with mnemonic', () => {
    const wallet = createNewWallet('test-password');
    
    expect(wallet.address).toMatch(/^0x[a-fA-F0-9]{40}$/);
    expect(wallet.encryptedMnemonic).toBeDefined();
    expect(wallet.encryptedPrivateKey).toBeDefined();
  });

  test('should encrypt and decrypt private key correctly', () => {
    const privateKey = '0x1234567890abcdef';
    const password = 'secure-password';
    
    const encrypted = encrypt(privateKey, password);
    const decrypted = decrypt(encrypted, password);
    
    expect(decrypted).toBe(privateKey);
  });
});
```

### Integration Tests
```typescript
// API Integration Tests
describe('Token Purchase API', () => {
  test('should create purchase record successfully', async () => {
    const purchaseData = {
      walletAddress: '0x742d35Cc6635Bb0532BC0532BC0532BC0532BC05',
      symbol: 'BOMB',
      amount: '100',
      usdValue: 50.00
    };

    const response = await request(app)
      .post('/api/purchased-tokens')
      .send(purchaseData)
      .expect(201);

    expect(response.body.success).toBe(true);
    expect(response.body.data.symbol).toBe('BOMB');
  });
});
```

## Deployment Configuration

### Environment Variables
```bash
# Database Configuration
DATABASE_URL=postgresql://user:password@host:port/database
POSTGRES_PRISMA_URL=postgresql://user:password@host:port/database

# Blockchain Configuration
BSC_RPC_URL=https://bsc-dataseed1.binance.org/
BSC_TESTNET_RPC_URL=https://data-seed-prebsc-1-s1.binance.org:8545/

# API Keys
COINGECKO_API_KEY=your_coingecko_api_key
TELEGRAM_BOT_TOKEN=your_telegram_bot_token

# Security
SESSION_SECRET=your_session_secret
ENCRYPTION_KEY=your_encryption_key

# Application
NODE_ENV=production
PORT=5000
```

### Build Configuration
```javascript
// vite.config.ts
export default defineConfig({
  plugins: [react()],
  build: {
    target: 'es2020',
    rollupOptions: {
      output: {
        manualChunks: {
          vendor: ['react', 'react-dom'],
          web3: ['web3', 'bip39', '@scure/bip32'],
          ui: ['@radix-ui/react-dialog', '@radix-ui/react-toast']
        }
      }
    }
  },
  optimizeDeps: {
    include: ['web3', 'bip39']
  }
});
```

This technical overview demonstrates the sophisticated architecture and implementation details that make Alpa Wallet a professional-grade cryptocurrency application, showcasing modern development practices and secure blockchain integration.