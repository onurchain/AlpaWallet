# 🚀 Deployment Guide - Alpa Wallet

## Production Deployment

### Platform: Replit

Alpa Wallet is optimized for deployment on Replit, providing seamless integration with modern web application infrastructure.

```json
{
  "name": "alpa-wallet-production",
  "run": "npm run dev",
  "env": {
    "NODE_ENV": "production",
    "DATABASE_URL": "${DATABASE_URL}",
    "SESSION_SECRET": "${SESSION_SECRET}",
    "COINGECKO_API_KEY": "${COINGECKO_API_KEY}"
  }
}
```

### Environment Configuration

#### Required Environment Variables
```bash
# Database Configuration
DATABASE_URL=postgresql://username:password@host:port/database

# Security
SESSION_SECRET=your-secure-session-secret-min-32-chars
ENCRYPTION_KEY=your-aes-encryption-key-32-chars

# External APIs
COINGECKO_API_KEY=your-coingecko-api-key
BSC_RPC_URL=https://bsc-dataseed1.binance.org/

# Application Settings
NODE_ENV=production
PORT=5000
CORS_ORIGIN=https://your-domain.replit.app
```

#### Optional Environment Variables
```bash
# Monitoring & Analytics
SENTRY_DSN=your-sentry-dsn
ANALYTICS_KEY=your-analytics-key

# Telegram Bot (if using notification features)
TELEGRAM_BOT_TOKEN=your-telegram-bot-token
TELEGRAM_CHAT_ID=your-telegram-chat-id

# Email Services (future features)
SMTP_HOST=your-smtp-host
SMTP_USER=your-smtp-username
SMTP_PASS=your-smtp-password
```

### Database Setup

#### PostgreSQL Configuration
```sql
-- Required database extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pgcrypto";

-- Recommended indexes for performance
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_wallets_address ON wallets(address);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_wallets_user_id ON wallets(user_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_purchased_tokens_wallet ON purchased_tokens(wallet_address);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_wallet_id ON transactions(wallet_id);
CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_transactions_timestamp ON transactions(timestamp DESC);
```

#### Database Migration Commands
```bash
# Push schema changes to database
npm run db:push

# Seed initial data (if needed)
npm run db:seed

# Generate Drizzle schema
npm run db:generate
```

### Build Process

#### Production Build
```bash
# Install dependencies
npm ci --only=production

# Build frontend assets
npm run build

# Type checking
npm run type-check

# Start production server
npm start
```

#### Build Configuration (vite.config.ts)
```typescript
export default defineConfig({
  plugins: [react()],
  build: {
    target: 'es2020',
    outDir: 'dist',
    sourcemap: false,
    minify: 'terser',
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
  server: {
    port: 5000,
    host: '0.0.0.0'
  }
});
```

## Security Considerations

### Production Security Checklist

#### Environment Security
- [ ] All sensitive keys stored in environment variables
- [ ] No hardcoded secrets in source code
- [ ] HTTPS enforced for all connections
- [ ] CORS properly configured for production domains
- [ ] Rate limiting implemented on API endpoints

#### Database Security
- [ ] Database connection uses SSL/TLS
- [ ] User permissions follow principle of least privilege  
- [ ] Regular automated backups configured
- [ ] Connection pooling optimized for production load
- [ ] Sensitive data encrypted at rest

#### Application Security
```typescript
// Security headers middleware
app.use((req, res, next) => {
  res.setHeader('X-Content-Type-Options', 'nosniff');
  res.setHeader('X-Frame-Options', 'DENY');
  res.setHeader('X-XSS-Protection', '1; mode=block');
  res.setHeader('Strict-Transport-Security', 'max-age=31536000; includeSubDomains');
  res.setHeader('Content-Security-Policy', "default-src 'self'; script-src 'self' 'unsafe-inline'");
  next();
});
```

### Monitoring & Logging

#### Error Tracking
```typescript
// Sentry integration for error monitoring
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1
});

// Custom error logging
export function logError(error: Error, context: any = {}) {
  Sentry.captureException(error, {
    tags: { component: 'wallet-service' },
    extra: context
  });
}
```

#### Performance Monitoring
```typescript
// Custom metrics collection
export const metrics = {
  transactionCount: 0,
  avgResponseTime: 0,
  activeUsers: new Set(),
  
  recordTransaction() {
    this.transactionCount++;
  },
  
  recordResponseTime(time: number) {
    this.avgResponseTime = (this.avgResponseTime + time) / 2;
  },
  
  addActiveUser(userId: string) {
    this.activeUsers.add(userId);
  }
};
```

## Scaling & Performance

### Horizontal Scaling
```typescript
// Load balancer configuration
const cluster = require('cluster');
const numCPUs = require('os').cpus().length;

if (cluster.isMaster && process.env.NODE_ENV === 'production') {
  console.log(`Master ${process.pid} is running`);
  
  // Fork workers
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }
  
  cluster.on('exit', (worker, code, signal) => {
    console.log(`Worker ${worker.process.pid} died`);
    cluster.fork();
  });
} else {
  require('./server/index.js');
}
```

### Database Optimization
```typescript
// Connection pooling configuration
const dbConfig = {
  connectionString: process.env.DATABASE_URL,
  ssl: process.env.NODE_ENV === 'production',
  pool: {
    min: 2,
    max: 10,
    acquireTimeoutMillis: 30000,
    createTimeoutMillis: 30000,
    destroyTimeoutMillis: 5000,
    idleTimeoutMillis: 30000,
    reapIntervalMillis: 1000
  }
};
```

### Caching Strategy
```typescript
// Redis cache configuration (if needed)
import Redis from 'ioredis';

const redis = new Redis({
  host: process.env.REDIS_HOST || 'localhost',
  port: parseInt(process.env.REDIS_PORT || '6379'),
  password: process.env.REDIS_PASSWORD,
  retryDelayOnFailover: 100,
  maxRetriesPerRequest: 3
});

// Cache token prices for 2 minutes
export async function getCachedTokenPrices() {
  const cached = await redis.get('token-prices');
  if (cached) {
    return JSON.parse(cached);
  }
  
  const prices = await fetchTokenPrices();
  await redis.setex('token-prices', 120, JSON.stringify(prices));
  return prices;
}
```

## Backup & Recovery

### Database Backup Strategy
```bash
#!/bin/bash
# Daily database backup script

DB_URL=$DATABASE_URL
BACKUP_DIR="/backups"
DATE=$(date +%Y%m%d_%H%M%S)

# Create backup
pg_dump $DB_URL > "$BACKUP_DIR/wallet_backup_$DATE.sql"

# Compress backup
gzip "$BACKUP_DIR/wallet_backup_$DATE.sql"

# Keep only last 30 days of backups
find $BACKUP_DIR -name "wallet_backup_*.sql.gz" -mtime +30 -delete
```

### Application State Backup
```typescript
// Backup critical application data
export async function createSystemBackup() {
  const backup = {
    timestamp: new Date().toISOString(),
    wallets: await db.query.wallets.findMany(),
    purchasedTokens: await db.query.purchasedTokens.findMany(),
    customTokens: await db.query.customTokens.findMany(),
    version: process.env.npm_package_version
  };
  
  const backupData = JSON.stringify(backup, null, 2);
  const filename = `system_backup_${Date.now()}.json`;
  
  // Store backup securely
  await storeBackup(filename, backupData);
  return filename;
}
```

## Health Checks & Monitoring

### Application Health Endpoint
```typescript
app.get('/health', async (req, res) => {
  const health = {
    status: 'healthy',
    timestamp: new Date().toISOString(),
    version: process.env.npm_package_version,
    uptime: process.uptime(),
    memory: process.memoryUsage(),
    checks: {
      database: false,
      blockchain: false,
      external_apis: false
    }
  };
  
  try {
    // Database check
    await db.raw('SELECT 1');
    health.checks.database = true;
    
    // Blockchain connectivity check  
    const web3 = new Web3(process.env.BSC_RPC_URL);
    await web3.eth.getBlockNumber();
    health.checks.blockchain = true;
    
    // External API check
    const response = await fetch('https://api.coingecko.com/api/v3/ping');
    health.checks.external_apis = response.ok;
    
  } catch (error) {
    health.status = 'degraded';
    console.error('Health check failed:', error);
  }
  
  const statusCode = health.status === 'healthy' ? 200 : 503;
  res.status(statusCode).json(health);
});
```

### Automated Monitoring Alerts
```typescript
// Automated alerting system
export class MonitoringService {
  private static alertThresholds = {
    responseTime: 5000, // 5 seconds
    errorRate: 0.05,    // 5%
    memoryUsage: 0.9    // 90%
  };
  
  static async checkSystemHealth() {
    const metrics = await this.collectMetrics();
    
    if (metrics.avgResponseTime > this.alertThresholds.responseTime) {
      await this.sendAlert('High response time detected');
    }
    
    if (metrics.errorRate > this.alertThresholds.errorRate) {
      await this.sendAlert('High error rate detected');  
    }
    
    if (metrics.memoryUsage > this.alertThresholds.memoryUsage) {
      await this.sendAlert('High memory usage detected');
    }
  }
  
  private static async sendAlert(message: string) {
    // Send to monitoring service (PagerDuty, Slack, etc.)
    console.error(`ALERT: ${message}`);
  }
}
```

## Troubleshooting

### Common Issues & Solutions

#### Database Connection Issues
```bash
# Check database connectivity
psql $DATABASE_URL -c "SELECT version();"

# Reset connection pool
npm run db:reset-pool

# Check for blocking queries
SELECT * FROM pg_stat_activity WHERE state = 'active';
```

#### Memory Issues
```bash
# Check memory usage
node --inspect server/index.js

# Enable garbage collection logging
node --trace-gc server/index.js

# Analyze heap dumps
node --heap-prof server/index.js
```

#### Performance Issues
```typescript
// Performance profiling middleware
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = Date.now() - start;
    if (duration > 1000) { // Log slow requests
      console.log(`Slow request: ${req.method} ${req.path} - ${duration}ms`);
    }
  });
  
  next();
});
```

This deployment guide ensures Alpa Wallet runs reliably in production with proper security, monitoring, and scalability considerations.