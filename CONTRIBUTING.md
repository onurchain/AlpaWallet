# 🤝 Contributing Guidelines - Alpa Wallet

## Overview

Thank you for your interest in Alpa Wallet! This document outlines how you can contribute to the project's development and improvement.

**Important Note**: This is a proprietary software project. While we welcome feedback and suggestions, please note that this repository is primarily for demonstration and portfolio purposes.

## 📋 Types of Contributions

### 🐛 Bug Reports
If you discover a bug or issue:

1. **Check Existing Issues**: Search existing issues to avoid duplicates
2. **Provide Detailed Information**:
   - Clear description of the problem
   - Steps to reproduce the issue
   - Expected vs actual behavior
   - Browser/device information
   - Screenshots if applicable

```markdown
## Bug Report Template

**Description**
A clear description of what the bug is.

**To Reproduce**
Steps to reproduce the behavior:
1. Go to '...'
2. Click on '....'
3. Scroll down to '....'
4. See error

**Expected Behavior**
A clear description of what you expected to happen.

**Screenshots**
If applicable, add screenshots to help explain your problem.

**Environment**
- Browser: [e.g. Chrome, Safari]
- Version: [e.g. 22]
- Device: [e.g. iPhone6, Desktop]
```

### 💡 Feature Suggestions
We welcome suggestions for new features:

1. **Use Feature Request Template**
2. **Explain the Use Case**: Why would this feature be valuable?
3. **Provide Examples**: How would it work?
4. **Consider Alternatives**: What alternatives have you considered?

```markdown
## Feature Request Template

**Is your feature request related to a problem?**
A clear description of what the problem is.

**Describe the solution you'd like**
A clear description of what you want to happen.

**Describe alternatives you've considered**
Alternative solutions or features you've considered.

**Additional context**
Add any other context or screenshots about the feature request.
```

### 📚 Documentation Improvements
Documentation contributions are always welcome:

- **Technical Documentation**: API documentation, code comments
- **User Guides**: Setup instructions, usage examples
- **Developer Guides**: Architecture explanations, contribution guides

## 🛠️ Development Setup

### Prerequisites
```bash
# Required software
- Node.js 18+ 
- npm or yarn
- PostgreSQL 14+
- Git

# Recommended tools
- VS Code with TypeScript extension
- Postman for API testing
- pgAdmin for database management
```

### Local Development Setup
```bash
# Clone the repository (if you have access)
git clone https://github.com/your-username/alpa-wallet.git
cd alpa-wallet

# Install dependencies
npm install

# Set up environment variables
cp .env.example .env
# Edit .env with your configuration

# Set up database
npm run db:push
npm run db:seed

# Start development server
npm run dev
```

### Environment Configuration
```bash
# .env.example template
DATABASE_URL=postgresql://username:password@localhost:5432/alpa_wallet
BSC_RPC_URL=https://bsc-dataseed1.binance.org/
COINGECKO_API_KEY=your_api_key_here
SESSION_SECRET=your_session_secret_here
NODE_ENV=development
PORT=5000
```

## 🔧 Development Guidelines

### Code Standards

#### TypeScript Best Practices
```typescript
// Use proper typing
interface TokenData {
  symbol: string;
  name: string;
  address: string;
  decimals: number;
  balance: string;
  usdValue: number;
}

// Prefer function declarations for components
export function TokenCard({ token }: { token: TokenData }) {
  return (
    <Card className="p-4">
      {/* Component content */}
    </Card>
  );
}

// Use proper error handling
try {
  const result = await riskyOperation();
  return result;
} catch (error) {
  console.error('Operation failed:', error);
  throw new Error('Detailed error message');
}
```

#### React Component Guidelines
```typescript
// Component structure
export function ComponentName({ prop1, prop2 }: ComponentProps) {
  // 1. Hooks at the top
  const [state, setState] = useState(initialValue);
  const { data, loading } = useQuery(queryOptions);
  
  // 2. Event handlers
  const handleClick = useCallback(() => {
    // Handler logic
  }, [dependencies]);
  
  // 3. Effects
  useEffect(() => {
    // Effect logic
  }, [dependencies]);
  
  // 4. Early returns
  if (loading) return <LoadingSpinner />;
  if (!data) return <EmptyState />;
  
  // 5. Main render
  return (
    <div className="component-container">
      {/* JSX content */}
    </div>
  );
}
```

#### CSS/Styling Guidelines
```typescript
// Use Tailwind CSS classes consistently
<div className="flex items-center justify-between p-4 bg-card rounded-lg border">
  <div className="flex items-center space-x-3">
    <Avatar className="h-10 w-10">
      <AvatarImage src={logoUrl} />
      <AvatarFallback>{symbol}</AvatarFallback>
    </Avatar>
    <div>
      <p className="font-medium text-foreground">{name}</p>
      <p className="text-sm text-muted-foreground">{symbol}</p>
    </div>
  </div>
</div>
```

#### Database Schema Guidelines
```typescript
// Use Drizzle ORM patterns
export const tableName = pgTable('table_name', {
  id: serial('id').primaryKey(),
  name: text('name').notNull(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
  updatedAt: timestamp('updated_at').defaultNow().notNull()
});

// Always include validation schemas
export const tableInsertSchema = createInsertSchema(tableName, {
  name: (schema) => schema.min(1, "Name is required").max(255)
});
```

### Testing Guidelines

#### Unit Tests
```typescript
// Use descriptive test names
describe('TokenService', () => {
  describe('calculateBalance', () => {
    it('should calculate USD balance correctly for valid token data', () => {
      const token = { balance: '100', price: 1.50 };
      const result = TokenService.calculateBalance(token);
      expect(result).toBe(150);
    });
    
    it('should handle invalid balance gracefully', () => {
      const token = { balance: 'invalid', price: 1.50 };
      const result = TokenService.calculateBalance(token);
      expect(result).toBe(0);
    });
  });
});
```

#### Integration Tests
```typescript
// Test API endpoints
describe('POST /api/purchase-token', () => {
  it('should create token purchase successfully', async () => {
    const purchaseData = {
      amount: '100',
      paymentMethod: 'usdt',
      walletAddress: '0x742d35Cc6635Bb0532BC0532BC0532BC0532BC05'
    };
    
    const response = await request(app)
      .post('/api/purchase-token')
      .send(purchaseData)
      .expect(201);
      
    expect(response.body.success).toBe(true);
    expect(response.body.data.tokenAmount).toBe('800');
  });
});
```

## 📋 Pull Request Process

### Before Submitting
1. **Fork the Repository** (if you have access)
2. **Create Feature Branch**: `git checkout -b feature/your-feature-name`
3. **Make Changes**: Follow the coding standards
4. **Test Thoroughly**: Ensure all tests pass
5. **Update Documentation**: If needed
6. **Commit Changes**: Use clear commit messages

### Commit Message Guidelines
```bash
# Format: type(scope): description

# Types:
feat: new feature
fix: bug fix
docs: documentation changes
style: formatting changes
refactor: code refactoring
test: adding tests
chore: maintenance tasks

# Examples:
feat(wallet): add token purchase functionality
fix(auth): resolve session timeout issue
docs(api): update endpoint documentation
refactor(hooks): simplify wallet state management
```

### Pull Request Template
```markdown
## Description
Brief description of changes made.

## Type of Change
- [ ] Bug fix (non-breaking change which fixes an issue)
- [ ] New feature (non-breaking change which adds functionality)
- [ ] Breaking change (fix or feature that would cause existing functionality to not work as expected)
- [ ] This change requires a documentation update

## Testing
- [ ] Tests pass locally
- [ ] Added tests for new functionality
- [ ] Tested on multiple browsers/devices

## Screenshots
If applicable, add screenshots of the changes.

## Checklist
- [ ] My code follows the style guidelines
- [ ] I have performed a self-review
- [ ] I have commented my code, particularly in hard-to-understand areas
- [ ] I have made corresponding changes to the documentation
```

## 🔒 Security Considerations

### Security Review Process
All contributions undergo security review:

1. **Code Analysis**: Automated security scanning
2. **Dependency Review**: Check for vulnerable packages
3. **Cryptographic Review**: Verify encryption implementations
4. **Access Control**: Ensure proper permission handling

### Security Guidelines
```typescript
// Never log sensitive information
console.log('User data:', { 
  userId: user.id, 
  // Don't log: privateKey, mnemonic, passwords
});

// Always validate input
const validateAddress = (address: string): boolean => {
  return /^0x[a-fA-F0-9]{40}$/.test(address);
};

// Use environment variables for secrets
const apiKey = process.env.API_KEY;
if (!apiKey) {
  throw new Error('API key is required');
}
```

## 📞 Communication Channels

### Getting Help
- **Documentation**: Check existing documentation first
- **Issues**: Create an issue for bugs or feature requests
- **Discussions**: Use GitHub Discussions for general questions

### Response Times
- **Bug Reports**: 1-3 business days
- **Feature Requests**: 1-2 weeks
- **Pull Reviews**: 3-5 business days
- **Security Issues**: 24 hours (high priority)

## 🏆 Recognition

### Contributors
We recognize valuable contributors through:
- **Contributor Credits**: Listed in project documentation
- **Achievement Badges**: GitHub profile recognition
- **Reference Letters**: Professional recommendations (for significant contributions)

### Contribution Types
- 🐛 **Bug Hunters**: Finding and reporting issues
- 💡 **Feature Architects**: Suggesting valuable enhancements  
- 📚 **Documentation Writers**: Improving project documentation
- 🔧 **Code Contributors**: Implementing features and fixes
- 🧪 **Quality Assurance**: Testing and validation

## 📜 Code of Conduct

### Our Standards
- **Respectful Communication**: Professional and courteous interactions
- **Constructive Feedback**: Focus on improvement, not criticism
- **Collaborative Spirit**: Work together toward common goals
- **Inclusive Environment**: Welcome all backgrounds and experience levels

### Enforcement
Violations of the code of conduct will result in:
1. **Warning**: First offense, private discussion
2. **Temporary Ban**: Repeated violations, limited access
3. **Permanent Ban**: Severe or persistent violations

---

Thank you for your interest in contributing to Alpa Wallet! Together, we can build a better cryptocurrency wallet experience for everyone.

**Note**: This project is proprietary software. All contributions are subject to the project license and contributor agreement.