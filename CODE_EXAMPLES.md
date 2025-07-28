# 💻 Code Examples - Alpa Wallet

## React Hook Implementation

### Custom Wallet Hook
```typescript
// hooks/useWallet.tsx
import { useState, useCallback, useEffect } from 'react';
import { Web3 } from 'web3';
import type { WalletState, Token } from '../types';

export function useWallet() {
  const [walletState, setWalletState] = useState<WalletState>({
    wallets: [],
    activeWallet: null,
    tokens: [],
    customTokens: [],
    isLoading: false,
    totalBalanceUsd: 0
  });

  // Initialize Web3 connection
  const web3 = new Web3('https://bsc-dataseed1.binance.org/');

  const syncTokenBalances = useCallback(async (walletAddress: string) => {
    setWalletState(prev => ({ ...prev, isLoading: true }));
    
    try {
      const balancePromises = walletState.tokens.map(async (token) => {
        if (token.symbol === 'BNB') {
          // Native BNB balance
          const balance = await web3.eth.getBalance(walletAddress);
          return {
            ...token,
            balance: web3.utils.fromWei(balance, 'ether'),
            usdValue: parseFloat(web3.utils.fromWei(balance, 'ether')) * token.price
          };
        } else {
          // ERC-20 token balance
          const contract = new web3.eth.Contract(ERC20_ABI, token.address);
          const balance = await contract.methods.balanceOf(walletAddress).call();
          const decimals = await contract.methods.decimals().call();
          const formattedBalance = balance / Math.pow(10, decimals);
          
          return {
            ...token,
            balance: formattedBalance.toString(),
            usdValue: formattedBalance * token.price
          };
        }
      });

      const updatedTokens = await Promise.all(balancePromises);
      const totalBalance = updatedTokens.reduce((sum, token) => sum + token.usdValue, 0);

      setWalletState(prev => ({
        ...prev,
        tokens: updatedTokens,
        totalBalanceUsd: totalBalance,
        isLoading: false
      }));
    } catch (error) {
      console.error('Balance sync failed:', error);
      setWalletState(prev => ({ ...prev, isLoading: false }));
    }
  }, [walletState.tokens, web3]);

  return {
    walletState,
    syncTokenBalances,
    createWallet,
    importWallet,
    sendTransaction
  };
}
```

### Token Purchase Component
```typescript
// components/TokenPurchaseScreen.tsx
import { useState } from 'react';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { Button } from '@/components/ui/button';
import { Input } from '@/components/ui/input';
import { Card, CardContent, CardHeader, CardTitle } from '@/components/ui/card';

interface PurchaseData {
  amount: string;
  paymentMethod: 'bnb' | 'usdt';
  walletAddress: string;
}

export function TokenPurchaseScreen() {
  const [purchaseAmount, setPurchaseAmount] = useState('');
  const [selectedPayment, setSelectedPayment] = useState<'bnb' | 'usdt'>('usdt');
  const queryClient = useQueryClient();

  const purchaseMutation = useMutation({
    mutationFn: async (data: PurchaseData) => {
      const response = await fetch('/api/purchase-token', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(data)
      });
      
      if (!response.ok) {
        throw new Error('Purchase failed');
      }
      
      return response.json();
    },
    onSuccess: () => {
      // Invalidate wallet balance to trigger refresh
      queryClient.invalidateQueries({ queryKey: ['wallet-balance'] });
      toast.success('Token purchase successful!');
    },
    onError: (error) => {
      toast.error(`Purchase failed: ${error.message}`);
    }
  });

  const handlePurchase = () => {
    if (!purchaseAmount || parseFloat(purchaseAmount) <= 0) {
      toast.error('Please enter a valid amount');
      return;
    }

    purchaseMutation.mutate({
      amount: purchaseAmount,
      paymentMethod: selectedPayment,
      walletAddress: activeWallet.address
    });
  };

  return (
    <Card className="max-w-md mx-auto">
      <CardHeader>
        <CardTitle className="text-center">Purchase BOMB Tokens</CardTitle>
      </CardHeader>
      <CardContent className="space-y-4">
        <div>
          <label className="text-sm font-medium">Amount (USDT)</label>
          <Input
            type="number"
            value={purchaseAmount}
            onChange={(e) => setPurchaseAmount(e.target.value)}
            placeholder="Enter amount"
            min="1"
            step="0.01"
          />
        </div>
        
        <div>
          <label className="text-sm font-medium">Payment Method</label>
          <div className="flex gap-2 mt-2">
            <Button
              variant={selectedPayment === 'usdt' ? 'default' : 'outline'}
              onClick={() => setSelectedPayment('usdt')}
              className="flex-1"
            >
              USDT
            </Button>
            <Button
              variant={selectedPayment === 'bnb' ? 'default' : 'outline'}
              onClick={() => setSelectedPayment('bnb')}
              className="flex-1"
            >
              BNB
            </Button>
          </div>
        </div>

        <Button
          onClick={handlePurchase}
          disabled={purchaseMutation.isPending}
          className="w-full"
        >
          {purchaseMutation.isPending ? 'Processing...' : 'Purchase Tokens'}
        </Button>
      </CardContent>
    </Card>
  );
}
```

## Backend API Implementation

### Express Route Handler
```typescript
// server/routes.ts
import express from 'express';
import { z } from 'zod';
import { db } from '../db';
import { purchasedTokens, wallets } from '../shared/schema';

const app = express();

// Validation schema
const purchaseSchema = z.object({
  amount: z.string().refine(val => !isNaN(parseFloat(val)) && parseFloat(val) > 0),
  paymentMethod: z.enum(['bnb', 'usdt']),
  walletAddress: z.string().regex(/^0x[a-fA-F0-9]{40}$/)
});

// Token purchase endpoint
app.post('/api/purchase-token', async (req, res) => {
  try {
    // Validate request body
    const validatedData = purchaseSchema.parse(req.body);
    const { amount, paymentMethod, walletAddress } = validatedData;

    // Verify wallet exists
    const wallet = await db.query.wallets.findFirst({
      where: eq(wallets.address, walletAddress)
    });

    if (!wallet) {
      return res.status(404).json({ error: 'Wallet not found' });
    }

    // Calculate token amount (example: 1 USDT = 8 BOMB tokens)
    const tokenAmount = (parseFloat(amount) * 8).toString();
    const usdValue = parseFloat(amount);

    // Create purchase record
    const [newPurchase] = await db.insert(purchasedTokens).values({
      walletAddress,
      symbol: 'BOMB',
      amount: tokenAmount,
      usdValue: usdValue.toString(),
      paymentMethod,
      status: 'confirmed',
      transactionHash: generateTransactionHash(),
      createdAt: new Date()
    }).returning();

    // Log the purchase
    console.log(`Token purchase: ${tokenAmount} BOMB for ${amount} ${paymentMethod.toUpperCase()}`);

    res.status(201).json({
      success: true,
      data: {
        id: newPurchase.id,
        tokenAmount,
        usdValue,
        transactionHash: newPurchase.transactionHash
      }
    });

  } catch (error) {
    if (error instanceof z.ZodError) {
      return res.status(400).json({
        error: 'Validation failed',
        details: error.errors
      });
    }

    console.error('Purchase error:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

// Get purchased tokens for wallet
app.get('/api/purchased-tokens/:address', async (req, res) => {
  try {
    const { address } = req.params;

    if (!address.match(/^0x[a-fA-F0-9]{40}$/)) {
      return res.status(400).json({ error: 'Invalid wallet address' });
    }

    const purchases = await db.query.purchasedTokens.findMany({
      where: eq(purchasedTokens.walletAddress, address),
      orderBy: desc(purchasedTokens.createdAt)
    });

    const totalTokens = purchases.reduce((sum, purchase) => {
      return sum + parseFloat(purchase.amount);
    }, 0);

    res.json({
      purchases,
      totalTokens: totalTokens.toString(),
      count: purchases.length
    });

  } catch (error) {
    console.error('Error fetching purchases:', error);
    res.status(500).json({ error: 'Internal server error' });
  }
});

function generateTransactionHash(): string {
  return '0x' + Array(64).fill(0).map(() => Math.floor(Math.random() * 16).toString(16)).join('');
}

export default app;
```

## Database Schema

### Drizzle ORM Schema
```typescript
// shared/schema.ts
import { pgTable, serial, text, timestamp, decimal, boolean, integer } from 'drizzle-orm/pg-core';
import { createInsertSchema, createSelectSchema } from 'drizzle-zod';
import { z } from 'zod';

// Users table
export const users = pgTable('users', {
  id: serial('id').primaryKey(),
  username: text('username').notNull().unique(),
  email: text('email').unique(),
  passwordHash: text('password_hash').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  lastLoginAt: timestamp('last_login_at')
});

// Wallets table
export const wallets = pgTable('wallets', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  address: text('address').notNull().unique(),
  encryptedKey: text('encrypted_key').notNull(),
  hasBackedUp: boolean('has_backed_up').default(false).notNull(),
  userId: integer('user_id').references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull()
});

// Custom tokens table
export const customTokens = pgTable('custom_tokens', {
  id: serial('id').primaryKey(),
  symbol: text('symbol').notNull(),
  name: text('name').notNull(),
  address: text('address').notNull(),
  decimals: integer('decimals').notNull(),
  isCustom: boolean('is_custom').default(true).notNull(),
  userId: integer('user_id').references(() => users.id).notNull(),
  walletId: integer('wallet_id').references(() => wallets.id).notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull()
});

// Purchased tokens table
export const purchasedTokens = pgTable('purchased_tokens', {
  id: serial('id').primaryKey(),
  walletAddress: text('wallet_address').notNull(),
  symbol: text('symbol').notNull(),
  amount: text('amount').notNull(),
  usdValue: decimal('usd_value', { precision: 10, scale: 2 }),
  paymentMethod: text('payment_method').notNull(),
  status: text('status').notNull().default('pending'),
  transactionHash: text('transaction_hash'),
  createdAt: timestamp('created_at').defaultNow().notNull()
});

// Transaction logs table
export const transactions = pgTable('transactions', {
  id: serial('id').primaryKey(),
  walletId: integer('wallet_id').references(() => wallets.id).notNull(),
  userId: integer('user_id').references(() => users.id).notNull(),
  type: text('type').notNull(), // 'send', 'receive', 'swap'
  symbol: text('symbol').notNull(),
  amount: text('amount').notNull(),
  toAddress: text('to_address'),
  fromAddress: text('from_address'),
  transactionHash: text('transaction_hash'),
  status: text('status').notNull().default('pending'),
  gasUsed: text('gas_used'),
  gasPrice: text('gas_price'),
  usdValue: decimal('usd_value', { precision: 10, scale: 2 }),
  timestamp: timestamp('timestamp').defaultNow().notNull()
});

// Validation schemas
export const walletsInsertSchema = createInsertSchema(wallets, {
  name: (schema) => schema.min(1, "Wallet name is required").max(50),
  address: (schema) => schema.regex(/^0x[a-fA-F0-9]{40}$/, "Invalid address format"),
  encryptedKey: (schema) => schema.min(1, "Encrypted key is required")
});

export const purchasedTokensInsertSchema = createInsertSchema(purchasedTokens, {
  amount: (schema) => schema.refine(val => !isNaN(parseFloat(val)) && parseFloat(val) > 0),
  walletAddress: (schema) => schema.regex(/^0x[a-fA-F0-9]{40}$/, "Invalid address"),
  symbol: (schema) => schema.min(1, "Symbol is required")
});

export const transactionsInsertSchema = createInsertSchema(transactions, {
  type: (schema) => schema.refine(val => ['send', 'receive', 'swap'].includes(val)),
  amount: (schema) => schema.refine(val => !isNaN(parseFloat(val)) && parseFloat(val) > 0)
});

// TypeScript types
export type User = z.infer<typeof createSelectSchema(users)>;
export type Wallet = z.infer<typeof createSelectSchema(wallets)>;
export type CustomToken = z.infer<typeof createSelectSchema(customTokens)>;
export type PurchasedToken = z.infer<typeof createSelectSchema(purchasedTokens)>;
export type Transaction = z.infer<typeof createSelectSchema(transactions)>;

export type WalletInsert = z.infer<typeof walletsInsertSchema>;
export type PurchasedTokenInsert = z.infer<typeof purchasedTokensInsertSchema>;
export type TransactionInsert = z.infer<typeof transactionsInsertSchema>;
```

## Blockchain Integration

### Web3 Utilities
```typescript
// lib/web3.ts
import { Web3 } from 'web3';
import type { TransactionReceipt, TransactionConfig } from 'web3-core';

const BSC_RPC_URL = 'https://bsc-dataseed1.binance.org/';
const BSC_TESTNET_RPC_URL = 'https://data-seed-prebsc-1-s1.binance.org:8545/';

// ERC-20 ABI for token interactions
const ERC20_ABI = [
  {
    "constant": true,
    "inputs": [{"name": "_owner", "type": "address"}],
    "name": "balanceOf",
    "outputs": [{"name": "balance", "type": "uint256"}],
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      {"name": "_to", "type": "address"},
      {"name": "_value", "type": "uint256"}
    ],
    "name": "transfer",
    "outputs": [{"name": "", "type": "bool"}],
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [],
    "name": "decimals",
    "outputs": [{"name": "", "type": "uint8"}],
    "type": "function"
  }
];

export class Web3Service {
  private web3: Web3;

  constructor(network: 'mainnet' | 'testnet' = 'mainnet') {
    const rpcUrl = network === 'mainnet' ? BSC_RPC_URL : BSC_TESTNET_RPC_URL;
    this.web3 = new Web3(rpcUrl);
  }

  // Get BNB balance
  async getBnbBalance(address: string): Promise<string> {
    try {
      const balance = await this.web3.eth.getBalance(address);
      return this.web3.utils.fromWei(balance, 'ether');
    } catch (error) {
      console.error('Error fetching BNB balance:', error);
      return '0';
    }
  }

  // Get ERC-20 token balance
  async getTokenBalance(tokenAddress: string, walletAddress: string): Promise<string> {
    try {
      const contract = new this.web3.eth.Contract(ERC20_ABI, tokenAddress);
      const balance = await contract.methods.balanceOf(walletAddress).call();
      const decimals = await contract.methods.decimals().call();
      
      return (parseFloat(balance.toString()) / Math.pow(10, decimals)).toString();
    } catch (error) {
      console.error(`Error fetching token balance for ${tokenAddress}:`, error);
      return '0';
    }
  }

  // Send BNB transaction
  async sendBnb(
    fromAddress: string,
    toAddress: string,
    amount: string,
    privateKey: string
  ): Promise<TransactionReceipt> {
    const transaction: TransactionConfig = {
      from: fromAddress,
      to: toAddress,
      value: this.web3.utils.toWei(amount, 'ether'),
      gas: 21000,
      gasPrice: await this.web3.eth.getGasPrice(),
      nonce: await this.web3.eth.getTransactionCount(fromAddress)
    };

    const signedTx = await this.web3.eth.accounts.signTransaction(transaction, privateKey);
    return await this.web3.eth.sendSignedTransaction(signedTx.rawTransaction!);
  }

  // Send ERC-20 token transaction
  async sendToken(
    tokenAddress: string,
    fromAddress: string,
    toAddress: string,
    amount: string,
    privateKey: string
  ): Promise<TransactionReceipt> {
    const contract = new this.web3.eth.Contract(ERC20_ABI, tokenAddress);
    const decimals = await contract.methods.decimals().call();
    const amountWei = (parseFloat(amount) * Math.pow(10, decimals)).toString();

    const transaction = contract.methods.transfer(toAddress, amountWei);
    const gas = await transaction.estimateGas({ from: fromAddress });
    const gasPrice = await this.web3.eth.getGasPrice();

    const transactionObject: TransactionConfig = {
      to: tokenAddress,
      data: transaction.encodeABI(),
      gas: gas.toString(),
      gasPrice: gasPrice.toString(),
      nonce: await this.web3.eth.getTransactionCount(fromAddress)
    };

    const signedTx = await this.web3.eth.accounts.signTransaction(transactionObject, privateKey);
    return await this.web3.eth.sendSignedTransaction(signedTx.rawTransaction!);
  }

  // Estimate gas for transaction
  async estimateGas(transaction: TransactionConfig): Promise<number> {
    try {
      return await this.web3.eth.estimateGas(transaction);
    } catch (error) {
      console.error('Gas estimation failed:', error);
      return 21000; // Default gas limit
    }
  }

  // Get current gas price
  async getGasPrice(): Promise<string> {
    return await this.web3.eth.getGasPrice();
  }

  // Validate address format
  isValidAddress(address: string): boolean {
    return this.web3.utils.isAddress(address);
  }

  // Convert Wei to Ether
  fromWei(value: string, unit: 'ether' | 'gwei' = 'ether'): string {
    return this.web3.utils.fromWei(value, unit);
  }

  // Convert Ether to Wei
  toWei(value: string, unit: 'ether' | 'gwei' = 'ether'): string {
    return this.web3.utils.toWei(value, unit);
  }
}

export const web3Service = new Web3Service();
```

## Security Implementation

### Encryption Utilities
```typescript
// lib/crypto.ts
import CryptoJS from 'crypto-js';

export class CryptoService {
  // Encrypt sensitive data using AES-256
  static encrypt(text: string, password: string): string {
    try {
      // Generate random salt
      const salt = CryptoJS.lib.WordArray.random(128/8);
      
      // Derive key using PBKDF2
      const key = CryptoJS.PBKDF2(password, salt, {
        keySize: 256/32,
        iterations: 10000,
        hasher: CryptoJS.algo.SHA256
      });
      
      // Generate random IV
      const iv = CryptoJS.lib.WordArray.random(128/8);
      
      // Encrypt the text
      const encrypted = CryptoJS.AES.encrypt(text, key, {
        iv: iv,
        padding: CryptoJS.pad.Pkcs7,
        mode: CryptoJS.mode.CBC
      });
      
      // Combine salt + iv + encrypted text
      const result = salt.toString() + iv.toString() + encrypted.toString();
      return result;
    } catch (error) {
      console.error('Encryption failed:', error);
      throw new Error('Encryption failed');
    }
  }

  // Decrypt sensitive data
  static decrypt(encryptedText: string, password: string): string {
    try {
      // Extract salt (first 32 chars)
      const salt = CryptoJS.enc.Hex.parse(encryptedText.substr(0, 32));
      
      // Extract IV (next 32 chars)
      const iv = CryptoJS.enc.Hex.parse(encryptedText.substr(32, 32));
      
      // Extract encrypted data (remaining)
      const encrypted = encryptedText.substring(64);
      
      // Derive key using same parameters
      const key = CryptoJS.PBKDF2(password, salt, {
        keySize: 256/32,
        iterations: 10000,
        hasher: CryptoJS.algo.SHA256
      });
      
      // Decrypt the data
      const decrypted = CryptoJS.AES.decrypt(encrypted, key, {
        iv: iv,
        padding: CryptoJS.pad.Pkcs7,
        mode: CryptoJS.mode.CBC
      });
      
      return decrypted.toString(CryptoJS.enc.Utf8);
    } catch (error) {
      console.error('Decryption failed:', error);
      throw new Error('Decryption failed');
    }
  }

  // Generate secure random password
  static generateSecurePassword(length: number = 32): string {
    const charset = 'abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789!@#$%^&*';
    let password = '';
    
    for (let i = 0; i < length; i++) {
      const randomIndex = Math.floor(Math.random() * charset.length);
      password += charset[randomIndex];
    }
    
    return password;
  }

  // Hash password for storage
  static hashPassword(password: string): string {
    const salt = CryptoJS.lib.WordArray.random(128/8);
    const hash = CryptoJS.PBKDF2(password, salt, {
      keySize: 256/32,
      iterations: 100000
    });
    
    return salt.toString() + hash.toString();
  }

  // Verify password against hash
  static verifyPassword(password: string, hash: string): boolean {
    try {
      const salt = CryptoJS.enc.Hex.parse(hash.substr(0, 32));
      const originalHash = hash.substring(32);
      
      const testHash = CryptoJS.PBKDF2(password, salt, {
        keySize: 256/32,
        iterations: 100000
      }).toString();
      
      return testHash === originalHash;
    } catch (error) {
      return false;
    }
  }
}
```

These code examples showcase the professional-grade implementation patterns used throughout Alpa Wallet, demonstrating modern React development, secure blockchain integration, and robust backend architecture.