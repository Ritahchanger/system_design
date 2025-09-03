# Testing Strategies for Frontend Microservices

Comprehensive testing approach for distributed frontend architectures, covering unit, integration, contract, and end-to-end testing strategies.

## 🧪 Testing Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Testing Pyramid                         │
├─────────────────────────────────────────────────────────────┤
│  E2E Tests           │ Full user journeys, critical paths   │
├─────────────────────────────────────────────────────────────┤
│  Integration Tests   │ Service interactions, API contracts  │
├─────────────────────────────────────────────────────────────┤
│  Component Tests     │ Individual components, logic         │
├─────────────────────────────────────────────────────────────┤
│  Unit Tests          │ Functions, utilities, helpers        │
└─────────────────────────────────────────────────────────────┘
```

## 🔬 Unit Testing

### Service-Specific Unit Tests

```javascript
// user-service/src/components/UserProfile.test.js
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import { authProvider } from '@shared/auth';

// Mock shared dependencies
jest.mock('@shared/auth');

describe('UserProfile', () => {
  beforeEach(() => {
    authProvider.getUser.mockReturnValue({
      id: '123',
      name: 'John Doe',
      email: 'john@example.com'
    });
  });

  it('displays user information correctly', () => {
    render(<UserProfile />);
    
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('handles profile update', async () => {
    const mockUpdateProfile = jest.fn().mockResolvedValue({ success: true });
    authProvider.updateProfile = mockUpdateProfile;

    render(<UserProfile />);
    
    const nameInput = screen.getByLabelText('Name');
    fireEvent.change(nameInput, { target: { value: 'Jane Doe' } });
    fireEvent.click(screen.getByText('Save'));

    await waitFor(() => {
      expect(mockUpdateProfile).toHaveBeenCalledWith({
        name: 'Jane Doe'
      });
    });
  });
});
```

### Shared Utilities Testing

```javascript
// shared/src/utils/validation.test.js
import { validateEmail, validateRequired, createValidator } from './validation';

describe('Validation Utils', () => {
  describe('validateEmail', () => {
    it('validates correct email formats', () => {
      expect(validateEmail('user@example.com')).toBe(true);
      expect(validateEmail('test.email+tag@domain.co.uk')).toBe(true);
    });

    it('rejects invalid email formats', () => {
      expect(validateEmail('invalid')).toBe(false);
      expect(validateEmail('@example.com')).toBe(false);
      expect(validateEmail('user@')).toBe(false);
    });
  });

  describe('createValidator', () => {
    it('creates validator with multiple rules', () => {
      const validator = createValidator([
        validateRequired('Name is required'),
        (value) => value.length >= 2 ? null : 'Too short'
      ]);

      expect(validator('')).toBe('Name is required');
      expect(validator('J')).toBe('Too short');
      expect(validator('John')).toBe(null);
    });
  });
});
```

## 🔗 Integration Testing

### Module Federation Integration

```javascript
// shell/src/__tests__/moduleIntegration.test.js
import { screen, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../test-utils';

// Mock module loading
const mockUserModule = {
  default: () => <div>User Module Loaded</div>
};

jest.mock('user_service/UserApp', () => mockUserModule);

describe('Module Integration', () => {
  it('loads and renders remote modules', async () => {
    renderWithProviders(<App />);
    
    // Navigate to user section
    fireEvent.click(screen.getByText('User Profile'));
    
    await waitFor(() => {
      expect(screen.getByText('User Module Loaded')).toBeInTheDocument();
    });
  });

  it('handles module loading errors gracefully', async () => {
    // Simulate module loading failure
    jest.mocked(mockUserModule).mockImplementationOnce(() => {
      throw new Error('Module load failed');
    });

    renderWithProviders(<App />);
    fireEvent.click(screen.getByText('User Profile'));

    await waitFor(() => {
      expect(screen.getByText('Service temporarily unavailable')).toBeInTheDocument();
    });
  });
});
```

### Cross-Service Communication Testing

```javascript
// shared/src/__tests__/eventBus.test.js
import { EventBus } from '../eventBus';

describe('EventBus Integration', () => {
  let eventBus;

  beforeEach(() => {
    eventBus = new EventBus();
  });

  it('handles cross-service events', (done) => {
    const eventData = { userId: '123', action: 'profile_updated' };
    
    eventBus.on('user:profile_updated', (data) => {
      expect(data).toEqual(eventData);
      done();
    });

    eventBus.emit('user:profile_updated', eventData);
  });

  it('removes event listeners properly', () => {
    const handler = jest.fn();
    
    eventBus.on('test:event', handler);
    eventBus.emit('test:event', {});
    expect(handler).toHaveBeenCalledTimes(1);

    eventBus.off('test:event', handler);
    eventBus.emit('test:event', {});
    expect(handler).toHaveBeenCalledTimes(1);
  });
});
```

## 📋 Contract Testing

### API Contract Tests with Pact

```javascript
// user-service/src/__tests__/contracts/userAPI.pact.test.js
import { Pact } from '@pact-foundation/pact';
import { UserService } from '../services/UserService';

describe('User API Contract', () => {
  let provider;
  let userService;

  beforeAll(() => {
    provider = new Pact({
      consumer: 'user-frontend',
      provider: 'user-api',
      port: 1234
    });

    userService = new UserService('http://localhost:1234');
  });

  afterAll(() => provider.finalize());

  describe('get user profile', () => {
    beforeEach(() => {
      const expectedRequest = {
        method: 'GET',
        path: '/api/users/123',
        headers: {
          'Authorization': 'Bearer token123'
        }
      };

      const expectedResponse = {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          id: '123',
          name: 'John Doe',
          email: 'john@example.com',
          createdAt: '2024-01-01T00:00:00Z'
        }
      };

      return provider
        .given('user exists')
        .uponReceiving('a request for user profile')
        .withRequest(expectedRequest)
        .willRespondWith(expectedResponse);
    });

    it('returns user profile data', async () => {
      const result = await userService.getProfile('123', 'token123');
      
      expect(result).toEqual({
        id: '123',
        name: 'John Doe',
        email: 'john@example.com',
        createdAt: '2024-01-01T00:00:00Z'
      });
    });
  });
});
```

### Module Contract Testing

```javascript
// shell/src/__tests__/contracts/moduleContracts.test.js
describe('Module Contracts', () => {
  it('user module exposes required interface', async () => {
    const { UserModule } = await import('user_service/UserApp');
    
    // Verify module structure
    expect(UserModule).toBeDefined();
    expect(typeof UserModule.mount).toBe('function');
    expect(typeof UserModule.unmount).toBe('function');
    
    // Test module lifecycle
    const container = document.createElement('div');
    await UserModule.mount(container, { userId: '123' });
    expect(container.children.length).toBeGreaterThan(0);
    
    UserModule.unmount(container);
    expect(container.children.length).toBe(0);
  });

  it('shared components maintain consistent API', () => {
    const { Button } = require('@shared/components');
    
    const requiredProps = ['children', 'onClick', 'variant', 'size'];
    const buttonProps = Object.keys(Button.propTypes || {});
    
    requiredProps.forEach(prop => {
      expect(buttonProps).toContain(prop);
    });
  });
});
```

## 🎭 End-to-End Testing

### Critical User Journey Tests

```javascript
// e2e/tests/userJourney.spec.js
import { test, expect } from '@playwright/test';

test.describe('User Journey', () => {
  test('complete user registration and profile setup', async ({ page }) => {
    // Start registration
    await page.goto('/');
    await page.click('text=Sign Up');
    
    // Fill registration form
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.click('[data-testid="register-button"]');
    
    // Wait for email verification
    await expect(page.locator('text=Check your email')).toBeVisible();
    
    // Simulate email verification (mock endpoint)
    await page.goto('/verify-email?token=mock-token');
    await expect(page.locator('text=Email verified')).toBeVisible();
    
    // Complete profile setup
    await page.click('text=Complete Profile');
    await page.fill('[data-testid="first-name"]', 'John');
    await page.fill('[data-testid="last-name"]', 'Doe');
    await page.selectOption('[data-testid="country"]', 'US');
    await page.click('[data-testid="save-profile"]');
    
    // Verify successful profile creation
    await expect(page.locator('[data-testid="profile-name"]')).toContainText('John Doe');
    await expect(page.locator('[data-testid="profile-email"]')).toContainText('test@example.com');
  });

  test('shopping cart functionality across modules', async ({ page }) => {
    await page.goto('/');
    
    // Login
    await page.fill('[data-testid="email"]', 'user@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    // Browse products (product catalog module)
    await page.click('text=Products');
    await page.click('[data-testid="product-card"]:first-child');
    await page.click('[data-testid="add-to-cart"]');
    
    // Verify cart update (cart module)
    await expect(page.locator('[data-testid="cart-count"]')).toContainText('1');
    
    // Proceed to checkout
    await page.click('[data-testid="cart-icon"]');
    await page.click('[data-testid="checkout-button"]');
    
    // Complete purchase
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="expiry"]', '12/25');
    await page.fill('[data-testid="cvc"]', '123');
    await page.click('[data-testid="pay-button"]');
    
    await expect(page.locator('text=Order confirmed')).toBeVisible();
  });
});
```

### Performance Testing

```javascript
// e2e/tests/performance.spec.js
import { test, expect } from '@playwright/test';

test.describe('Performance Tests', () => {
  test('module loading performance', async ({ page }) => {
    await page.goto('/');
    
    // Measure initial load time
    const navigationTiming = await page.evaluate(() => 
      JSON.stringify(performance.getEntriesByType('navigation'))
    );
    
    const navigation = JSON.parse(navigationTiming)[0];
    const loadTime = navigation.loadEventEnd - navigation.fetchStart;
    
    expect(loadTime).toBeLessThan(3000); // 3 seconds max
    
    // Measure module federation load
    const moduleLoadStart = Date.now();
    await page.click('text=User Profile');
    await page.waitForSelector('[data-testid="user-profile"]');
    const moduleLoadTime = Date.now() - moduleLoadStart;
    
    expect(moduleLoadTime).toBeLessThan(1000); // 1 second max for module load
  });

  test('memory usage during navigation', async ({ page }) => {
    await page.goto('/');
    
    // Get initial memory usage
    const initialMemory = await page.evaluate(() => {
      return performance.memory ? performance.memory.usedJSHeapSize : 0;
    });
    
    // Navigate through different modules
    const modules = ['products', 'profile', 'cart', 'settings'];
    
    for (const module of modules) {
      await page.click(`text=${module}`);
      await page.waitForTimeout(1000);
    }
    
    const finalMemory = await page.evaluate(() => {
      return performance.memory ? performance.memory.usedJSHeapSize : 0;
    });
    
    // Memory should not increase by more than 10MB
    const memoryIncrease = (finalMemory - initialMemory) / 1024 / 1024;
    expect(memoryIncrease).toBeLessThan(10);
  });
});
```

## 🛠️ Test Configuration

### Jest Configuration for Monorepo

```javascript
// jest.config.js
module.exports = {
  projects: [
    {
      displayName: 'shell',
      testMatch: ['<rootDir>/shell/src/**/*.test.{js,ts,tsx}'],
      setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
              moduleNameMapping: {
          '@shared/(.*)': '<rootDir>/shared/src/$1'
        }
      },
      {
        displayName: 'shared',
        testMatch: ['<rootDir>/shared/src/**/*.test.{js,ts}'],
        testEnvironment: 'jsdom'
      }
    ],
    coverageDirectory: '<rootDir>/coverage',
    collectCoverageFrom: [
      'src/**/*.{js,ts,tsx}',
      '!src/**/*.d.ts',
      '!src/**/index.{js,ts}'
    ],
    coverageThreshold: {
      global: {
        branches: 70,
        functions: 80,
        lines: 80,
        statements: 80
      }
    }
  };
```

### Test Utilities and Setup

```javascript
// jest.setup.js
import '@testing-library/jest-dom';
import { TextEncoder, TextDecoder } from 'util';

// Polyfills for Node.js environment
global.TextEncoder = TextEncoder;
global.TextDecoder = TextDecoder;

// Mock Module Federation
global.__webpack_require__ = {
  ensure: jest.fn((deps, callback) => callback()),
};

// Mock browser APIs
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// Mock IntersectionObserver
global.IntersectionObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}));
```

### Test Utilities

```javascript
// shared/src/test-utils/index.js
import React from 'react';
import { render } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from 'react-query';

// Test wrapper with providers
export function renderWithProviders(ui, {
  initialEntries = ['/'],
  queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  }),
  ...options
} = {}) {
  function Wrapper({ children }) {
    return (
      <QueryClientProvider client={queryClient}>
        <BrowserRouter>
          {children}
        </BrowserRouter>
      </QueryClientProvider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...options });
}

// Mock event bus for testing
export class MockEventBus {
  constructor() {
    this.listeners = new Map();
  }

  on(event, handler) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(handler);
  }

  emit(event, data) {
    const handlers = this.listeners.get(event) || [];
    handlers.forEach(handler => handler(data));
  }

  off(event, handler) {
    const handlers = this.listeners.get(event) || [];
    const index = handlers.indexOf(handler);
    if (index > -1) {
      handlers.splice(index, 1);
    }
  }

  clear() {
    this.listeners.clear();
  }
}

// API mocking utilities
export function createMockAPI() {
  return {
    get: jest.fn(),
    post: jest.fn(),
    put: jest.fn(),
    delete: jest.fn(),
  };
}
```

## 🎯 Visual Regression Testing

### Storybook Integration

```javascript
// .storybook/test-runner.js
module.exports = {
  async postRender(page, context) {
    const elementHandler = await page.$('#root');
    const innerHTML = await elementHandler.innerHTML();
    
    // Check for console errors
    const errors = await page.evaluate(() => {
      return window.__errors || [];
    });
    
    if (errors.length > 0) {
      throw new Error(`Console errors: ${errors.join(', ')}`);
    }

    // Visual regression testing
    await page.screenshot({
      path: `screenshots/${context.id}.png`,
      fullPage: true
    });
  },
};
```

### Component Visual Tests

```javascript
// shared/src/components/__tests__/Button.visual.test.js
import { test, expect } from '@playwright/test';

test.describe('Button Visual Tests', () => {
  test('renders all button variants correctly', async ({ page }) => {
    await page.goto('/storybook/button');
    
    // Test different variants
    const variants = ['primary', 'secondary', 'danger', 'outline'];
    
    for (const variant of variants) {
      await page.click(`[data-variant="${variant}"]`);
      await expect(page).toHaveScreenshot(`button-${variant}.png`);
    }
  });

  test('button states visual consistency', async ({ page }) => {
    await page.goto('/storybook/button');
    
    // Normal state
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-normal.png');
    
    // Hover state
    await page.hover('[data-testid="button"]');
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-hover.png');
    
    // Disabled state
    await page.click('[data-testid="toggle-disabled"]');
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-disabled.png');
  });
});
```

## 🔍 Testing Microservice Boundaries

### Boundary Testing

```javascript
// shell/src/__tests__/serviceBoundaries.test.js
describe('Service Boundaries', () => {
  it('services maintain independent state', async () => {
    const { UserService } = await import('user_service/UserApp');
    const { ProductService } = await import('product_service/ProductApp');
    
    // Mount both services
    const userContainer = document.createElement('div');
    const productContainer = document.createElement('div');
    
    await UserService.mount(userContainer, { userId: '123' });
    await ProductService.mount(productContainer, { categoryId: '456' });
    
    // Verify isolation
    expect(userContainer.querySelector('[data-service="user"]')).toBeTruthy();
    expect(productContainer.querySelector('[data-service="product"]')).toBeTruthy();
    
    // State changes in one service shouldn't affect another
    const userState = await UserService.getState();
    await ProductService.updateState({ selectedProduct: '789' });
    const userStateAfter = await UserService.getState();
    
    expect(userState).toEqual(userStateAfter);
  });

  it('services communicate only through defined interfaces', () => {
    const eventBus = new MockEventBus();
    
    // Register services with event bus
    const userService = new UserService(eventBus);
    const productService = new ProductService(eventBus);
    
    // Test allowed communication
    const mockHandler = jest.fn();
    productService.on('user:preferences_updated', mockHandler);
    
    userService.updatePreferences({ theme: 'dark' });
    
    expect(mockHandler).toHaveBeenCalledWith({
      userId: expect.any(String),
      preferences: { theme: 'dark' }
    });
  });
});
```

## 📊 Test Reporting and Analytics

### Custom Test Reporter

```javascript
// test-utils/customReporter.js
class MicroserviceTestReporter {
  constructor(globalConfig, options) {
    this.globalConfig = globalConfig;
    this.options = options;
    this.results = {
      services: {},
      integration: {},
      e2e: {}
    };
  }

  onRunStart() {
    console.log('🚀 Starting microservice test suite...');
  }

  onTestResult(test, testResult) {
    const serviceName = this.extractServiceName(test.path);
    
    if (!this.results.services[serviceName]) {
      this.results.services[serviceName] = {
        passed: 0,
        failed: 0,
        coverage: 0
      };
    }

    testResult.testResults.forEach(result => {
      if (result.status === 'passed') {
        this.results.services[serviceName].passed++;
      } else {
        this.results.services[serviceName].failed++;
      }
    });
  }

  onRunComplete() {
    this.generateReport();
  }

  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      summary: this.generateSummary(),
      services: this.results.services,
      recommendations: this.generateRecommendations()
    };

    console.table(this.results.services);
    
    // Save detailed report
    require('fs').writeFileSync(
      'test-results/microservice-report.json',
      JSON.stringify(report, null, 2)
    );
  }

  extractServiceName(testPath) {
    const match = testPath.match(/\/([^\/]+)\/src\//);
    return match ? match[1] : 'unknown';
  }

  generateSummary() {
    let totalPassed = 0;
    let totalFailed = 0;

    Object.values(this.results.services).forEach(service => {
      totalPassed += service.passed;
      totalFailed += service.failed;
    });

    return {
      total: totalPassed + totalFailed,
      passed: totalPassed,
      failed: totalFailed,
      success_rate: (totalPassed / (totalPassed + totalFailed)) * 100
    };
  }

  generateRecommendations() {
    const recommendations = [];
    
    Object.entries(this.results.services).forEach(([service, results]) => {
      const failureRate = results.failed / (results.passed + results.failed);
      
      if (failureRate > 0.1) {
        recommendations.push(`🔴 ${service}: High failure rate (${(failureRate * 100).toFixed(1)}%) - needs attention`);
      }
      
      if (results.coverage < 70) {
        recommendations.push(`📊 ${service}: Low coverage (${results.coverage}%) - add more tests`);
      }
    });

    return recommendations;
  }
}

module.exports = MicroserviceTestReporter;
```

## 🚀 CI/CD Testing Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
        
      - name: Start test services
        run: |
          docker-compose -f docker-compose.test.yml up -d
          npm run wait-for-services
          
      - name: Run integration tests
        run: npm run test:integration
        
      - name: Run contract tests
        run: npm run test:contracts

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
        
      - name: Install Playwright
        run: npx playwright install
        
      - name: Build and start application
        run: |
          npm run build:all
          npm run start:test &
          npm run wait-for-app
          
      - name: Run E2E tests
        run: npm run test:e2e
        
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: e2e-report
          path: playwright-report/
```

## 📋 Testing Best Practices

### Service-Specific Guidelines

1. **Unit Tests (70% of test suite)**
   - Test business logic in isolation
   - Mock external dependencies
   - Fast execution (< 1ms per test)
   - High code coverage (>80%)

2. **Integration Tests (20% of test suite)**
   - Test service interactions
   - Use real dependencies where possible
   - Focus on data flow and contracts
   - Medium execution time (< 100ms per test)

3. **E2E Tests (10% of test suite)**
   - Test critical user journeys
   - Use production-like environment
   - Focus on high-value scenarios
   - Slower execution acceptable

### Quality Gates

```javascript
// scripts/quality-gates.js
const fs = require('fs');

function checkQualityGates() {
  const coverage = JSON.parse(fs.readFileSync('coverage/coverage-summary.json'));
  const testResults = JSON.parse(fs.readFileSync('test-results.json'));
  
  const gates = {
    coverage: coverage.total.lines.pct >= 80,
    testSuccess: testResults.success >= 95,
    performance: testResults.avgDuration < 5000
  };
  
  const passed = Object.values(gates).every(gate => gate);
  
  if (!passed) {
    console.error('❌ Quality gates failed:', gates);
    process.exit(1);
  }
  
  console.log('✅ All quality gates passed');
}

checkQualityGates();
```

This comprehensive testing strategy ensures reliability, maintainability, and confidence in your frontend microservices architecture. The pyramid approach balances test coverage with execution speed, while the specialized tools handle the unique challenges of distributed frontend systems.
        '@shared/(.*)': '<rootDir>/shared/src/$1'
      }
    },
    {
      displayName: 'user-service',
      testMatch: ['<rootDir>/user-service/src/**/*.test.{js,ts,tsx}'],
      setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
      moduleNameMapping: {# Testing Strategies for Frontend Microservices

Comprehensive testing approach for distributed frontend architectures, covering unit, integration, contract, and end-to-end testing strategies.

## 🧪 Testing Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Testing Pyramid                         │
├─────────────────────────────────────────────────────────────┤
│  E2E Tests           │ Full user journeys, critical paths   │
├─────────────────────────────────────────────────────────────┤
│  Integration Tests   │ Service interactions, API contracts  │
├─────────────────────────────────────────────────────────────┤
│  Component Tests     │ Individual components, logic         │
├─────────────────────────────────────────────────────────────┤
│  Unit Tests          │ Functions, utilities, helpers        │
└─────────────────────────────────────────────────────────────┘
```

## 🔬 Unit Testing

### Service-Specific Unit Tests

```javascript
// user-service/src/components/UserProfile.test.js
import { render, screen, fireEvent, waitFor } from '@testing-library/react';
import { UserProfile } from './UserProfile';
import { authProvider } from '@shared/auth';

// Mock shared dependencies
jest.mock('@shared/auth');

describe('UserProfile', () => {
  beforeEach(() => {
    authProvider.getUser.mockReturnValue({
      id: '123',
      name: 'John Doe',
      email: 'john@example.com'
    });
  });

  it('displays user information correctly', () => {
    render(<UserProfile />);
    
    expect(screen.getByText('John Doe')).toBeInTheDocument();
    expect(screen.getByText('john@example.com')).toBeInTheDocument();
  });

  it('handles profile update', async () => {
    const mockUpdateProfile = jest.fn().mockResolvedValue({ success: true });
    authProvider.updateProfile = mockUpdateProfile;

    render(<UserProfile />);
    
    const nameInput = screen.getByLabelText('Name');
    fireEvent.change(nameInput, { target: { value: 'Jane Doe' } });
    fireEvent.click(screen.getByText('Save'));

    await waitFor(() => {
      expect(mockUpdateProfile).toHaveBeenCalledWith({
        name: 'Jane Doe'
      });
    });
  });
});
```

### Shared Utilities Testing

```javascript
// shared/src/utils/validation.test.js
import { validateEmail, validateRequired, createValidator } from './validation';

describe('Validation Utils', () => {
  describe('validateEmail', () => {
    it('validates correct email formats', () => {
      expect(validateEmail('user@example.com')).toBe(true);
      expect(validateEmail('test.email+tag@domain.co.uk')).toBe(true);
    });

    it('rejects invalid email formats', () => {
      expect(validateEmail('invalid')).toBe(false);
      expect(validateEmail('@example.com')).toBe(false);
      expect(validateEmail('user@')).toBe(false);
    });
  });

  describe('createValidator', () => {
    it('creates validator with multiple rules', () => {
      const validator = createValidator([
        validateRequired('Name is required'),
        (value) => value.length >= 2 ? null : 'Too short'
      ]);

      expect(validator('')).toBe('Name is required');
      expect(validator('J')).toBe('Too short');
      expect(validator('John')).toBe(null);
    });
  });
});
```

## 🔗 Integration Testing

### Module Federation Integration

```javascript
// shell/src/__tests__/moduleIntegration.test.js
import { screen, waitFor } from '@testing-library/react';
import { renderWithProviders } from '../test-utils';

// Mock module loading
const mockUserModule = {
  default: () => <div>User Module Loaded</div>
};

jest.mock('user_service/UserApp', () => mockUserModule);

describe('Module Integration', () => {
  it('loads and renders remote modules', async () => {
    renderWithProviders(<App />);
    
    // Navigate to user section
    fireEvent.click(screen.getByText('User Profile'));
    
    await waitFor(() => {
      expect(screen.getByText('User Module Loaded')).toBeInTheDocument();
    });
  });

  it('handles module loading errors gracefully', async () => {
    // Simulate module loading failure
    jest.mocked(mockUserModule).mockImplementationOnce(() => {
      throw new Error('Module load failed');
    });

    renderWithProviders(<App />);
    fireEvent.click(screen.getByText('User Profile'));

    await waitFor(() => {
      expect(screen.getByText('Service temporarily unavailable')).toBeInTheDocument();
    });
  });
});
```

### Cross-Service Communication Testing

```javascript
// shared/src/__tests__/eventBus.test.js
import { EventBus } from '../eventBus';

describe('EventBus Integration', () => {
  let eventBus;

  beforeEach(() => {
    eventBus = new EventBus();
  });

  it('handles cross-service events', (done) => {
    const eventData = { userId: '123', action: 'profile_updated' };
    
    eventBus.on('user:profile_updated', (data) => {
      expect(data).toEqual(eventData);
      done();
    });

    eventBus.emit('user:profile_updated', eventData);
  });

  it('removes event listeners properly', () => {
    const handler = jest.fn();
    
    eventBus.on('test:event', handler);
    eventBus.emit('test:event', {});
    expect(handler).toHaveBeenCalledTimes(1);

    eventBus.off('test:event', handler);
    eventBus.emit('test:event', {});
    expect(handler).toHaveBeenCalledTimes(1);
  });
});
```

## 📋 Contract Testing

### API Contract Tests with Pact

```javascript
// user-service/src/__tests__/contracts/userAPI.pact.test.js
import { Pact } from '@pact-foundation/pact';
import { UserService } from '../services/UserService';

describe('User API Contract', () => {
  let provider;
  let userService;

  beforeAll(() => {
    provider = new Pact({
      consumer: 'user-frontend',
      provider: 'user-api',
      port: 1234
    });

    userService = new UserService('http://localhost:1234');
  });

  afterAll(() => provider.finalize());

  describe('get user profile', () => {
    beforeEach(() => {
      const expectedRequest = {
        method: 'GET',
        path: '/api/users/123',
        headers: {
          'Authorization': 'Bearer token123'
        }
      };

      const expectedResponse = {
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          id: '123',
          name: 'John Doe',
          email: 'john@example.com',
          createdAt: '2024-01-01T00:00:00Z'
        }
      };

      return provider
        .given('user exists')
        .uponReceiving('a request for user profile')
        .withRequest(expectedRequest)
        .willRespondWith(expectedResponse);
    });

    it('returns user profile data', async () => {
      const result = await userService.getProfile('123', 'token123');
      
      expect(result).toEqual({
        id: '123',
        name: 'John Doe',
        email: 'john@example.com',
        createdAt: '2024-01-01T00:00:00Z'
      });
    });
  });
});
```

### Module Contract Testing

```javascript
// shell/src/__tests__/contracts/moduleContracts.test.js
describe('Module Contracts', () => {
  it('user module exposes required interface', async () => {
    const { UserModule } = await import('user_service/UserApp');
    
    // Verify module structure
    expect(UserModule).toBeDefined();
    expect(typeof UserModule.mount).toBe('function');
    expect(typeof UserModule.unmount).toBe('function');
    
    // Test module lifecycle
    const container = document.createElement('div');
    await UserModule.mount(container, { userId: '123' });
    expect(container.children.length).toBeGreaterThan(0);
    
    UserModule.unmount(container);
    expect(container.children.length).toBe(0);
  });

  it('shared components maintain consistent API', () => {
    const { Button } = require('@shared/components');
    
    const requiredProps = ['children', 'onClick', 'variant', 'size'];
    const buttonProps = Object.keys(Button.propTypes || {});
    
    requiredProps.forEach(prop => {
      expect(buttonProps).toContain(prop);
    });
  });
});
```

## 🎭 End-to-End Testing

### Critical User Journey Tests

```javascript
// e2e/tests/userJourney.spec.js
import { test, expect } from '@playwright/test';

test.describe('User Journey', () => {
  test('complete user registration and profile setup', async ({ page }) => {
    // Start registration
    await page.goto('/');
    await page.click('text=Sign Up');
    
    // Fill registration form
    await page.fill('[data-testid="email-input"]', 'test@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePass123!');
    await page.click('[data-testid="register-button"]');
    
    // Wait for email verification
    await expect(page.locator('text=Check your email')).toBeVisible();
    
    // Simulate email verification (mock endpoint)
    await page.goto('/verify-email?token=mock-token');
    await expect(page.locator('text=Email verified')).toBeVisible();
    
    // Complete profile setup
    await page.click('text=Complete Profile');
    await page.fill('[data-testid="first-name"]', 'John');
    await page.fill('[data-testid="last-name"]', 'Doe');
    await page.selectOption('[data-testid="country"]', 'US');
    await page.click('[data-testid="save-profile"]');
    
    // Verify successful profile creation
    await expect(page.locator('[data-testid="profile-name"]')).toContainText('John Doe');
    await expect(page.locator('[data-testid="profile-email"]')).toContainText('test@example.com');
  });

  test('shopping cart functionality across modules', async ({ page }) => {
    await page.goto('/');
    
    // Login
    await page.fill('[data-testid="email"]', 'user@example.com');
    await page.fill('[data-testid="password"]', 'password123');
    await page.click('[data-testid="login-button"]');
    
    // Browse products (product catalog module)
    await page.click('text=Products');
    await page.click('[data-testid="product-card"]:first-child');
    await page.click('[data-testid="add-to-cart"]');
    
    // Verify cart update (cart module)
    await expect(page.locator('[data-testid="cart-count"]')).toContainText('1');
    
    // Proceed to checkout
    await page.click('[data-testid="cart-icon"]');
    await page.click('[data-testid="checkout-button"]');
    
    // Complete purchase
    await page.fill('[data-testid="card-number"]', '4242424242424242');
    await page.fill('[data-testid="expiry"]', '12/25');
    await page.fill('[data-testid="cvc"]', '123');
    await page.click('[data-testid="pay-button"]');
    
    await expect(page.locator('text=Order confirmed')).toBeVisible();
  });
});
```

### Performance Testing

```javascript
// e2e/tests/performance.spec.js
import { test, expect } from '@playwright/test';

test.describe('Performance Tests', () => {
  test('module loading performance', async ({ page }) => {
    await page.goto('/');
    
    // Measure initial load time
    const navigationTiming = await page.evaluate(() => 
      JSON.stringify(performance.getEntriesByType('navigation'))
    );
    
    const navigation = JSON.parse(navigationTiming)[0];
    const loadTime = navigation.loadEventEnd - navigation.fetchStart;
    
    expect(loadTime).toBeLessThan(3000); // 3 seconds max
    
    // Measure module federation load
    const moduleLoadStart = Date.now();
    await page.click('text=User Profile');
    await page.waitForSelector('[data-testid="user-profile"]');
    const moduleLoadTime = Date.now() - moduleLoadStart;
    
    expect(moduleLoadTime).toBeLessThan(1000); // 1 second max for module load
  });

  test('memory usage during navigation', async ({ page }) => {
    await page.goto('/');
    
    // Get initial memory usage
    const initialMemory = await page.evaluate(() => {
      return performance.memory ? performance.memory.usedJSHeapSize : 0;
    });
    
    // Navigate through different modules
    const modules = ['products', 'profile', 'cart', 'settings'];
    
    for (const module of modules) {
      await page.click(`text=${module}`);
      await page.waitForTimeout(1000);
    }
    
    const finalMemory = await page.evaluate(() => {
      return performance.memory ? performance.memory.usedJSHeapSize : 0;
    });
    
    // Memory should not increase by more than 10MB
    const memoryIncrease = (finalMemory - initialMemory) / 1024 / 1024;
    expect(memoryIncrease).toBeLessThan(10);
  });
});
```

## 🛠️ Test Configuration

### Jest Configuration for Monorepo

```javascript
// jest.config.js
module.exports = {
  projects: [
    {
      displayName: 'shell',
      testMatch: ['<rootDir>/shell/src/**/*.test.{js,ts,tsx}'],
      setupFilesAfterEnv: ['<rootDir>/jest.setup.js'],
              moduleNameMapping: {
          '@shared/(.*)': '<rootDir>/shared/src/$1'
        }
      },
      {
        displayName: 'shared',
        testMatch: ['<rootDir>/shared/src/**/*.test.{js,ts}'],
        testEnvironment: 'jsdom'
      }
    ],
    coverageDirectory: '<rootDir>/coverage',
    collectCoverageFrom: [
      'src/**/*.{js,ts,tsx}',
      '!src/**/*.d.ts',
      '!src/**/index.{js,ts}'
    ],
    coverageThreshold: {
      global: {
        branches: 70,
        functions: 80,
        lines: 80,
        statements: 80
      }
    }
  };
```

### Test Utilities and Setup

```javascript
// jest.setup.js
import '@testing-library/jest-dom';
import { TextEncoder, TextDecoder } from 'util';

// Polyfills for Node.js environment
global.TextEncoder = TextEncoder;
global.TextDecoder = TextDecoder;

// Mock Module Federation
global.__webpack_require__ = {
  ensure: jest.fn((deps, callback) => callback()),
};

// Mock browser APIs
Object.defineProperty(window, 'matchMedia', {
  writable: true,
  value: jest.fn().mockImplementation(query => ({
    matches: false,
    media: query,
    onchange: null,
    addListener: jest.fn(),
    removeListener: jest.fn(),
    addEventListener: jest.fn(),
    removeEventListener: jest.fn(),
    dispatchEvent: jest.fn(),
  })),
});

// Mock IntersectionObserver
global.IntersectionObserver = jest.fn().mockImplementation(() => ({
  observe: jest.fn(),
  unobserve: jest.fn(),
  disconnect: jest.fn(),
}));
```

### Test Utilities

```javascript
// shared/src/test-utils/index.js
import React from 'react';
import { render } from '@testing-library/react';
import { BrowserRouter } from 'react-router-dom';
import { QueryClient, QueryClientProvider } from 'react-query';

// Test wrapper with providers
export function renderWithProviders(ui, {
  initialEntries = ['/'],
  queryClient = new QueryClient({
    defaultOptions: {
      queries: { retry: false },
      mutations: { retry: false }
    }
  }),
  ...options
} = {}) {
  function Wrapper({ children }) {
    return (
      <QueryClientProvider client={queryClient}>
        <BrowserRouter>
          {children}
        </BrowserRouter>
      </QueryClientProvider>
    );
  }

  return render(ui, { wrapper: Wrapper, ...options });
}

// Mock event bus for testing
export class MockEventBus {
  constructor() {
    this.listeners = new Map();
  }

  on(event, handler) {
    if (!this.listeners.has(event)) {
      this.listeners.set(event, []);
    }
    this.listeners.get(event).push(handler);
  }

  emit(event, data) {
    const handlers = this.listeners.get(event) || [];
    handlers.forEach(handler => handler(data));
  }

  off(event, handler) {
    const handlers = this.listeners.get(event) || [];
    const index = handlers.indexOf(handler);
    if (index > -1) {
      handlers.splice(index, 1);
    }
  }

  clear() {
    this.listeners.clear();
  }
}

// API mocking utilities
export function createMockAPI() {
  return {
    get: jest.fn(),
    post: jest.fn(),
    put: jest.fn(),
    delete: jest.fn(),
  };
}
```

## 🎯 Visual Regression Testing

### Storybook Integration

```javascript
// .storybook/test-runner.js
module.exports = {
  async postRender(page, context) {
    const elementHandler = await page.$('#root');
    const innerHTML = await elementHandler.innerHTML();
    
    // Check for console errors
    const errors = await page.evaluate(() => {
      return window.__errors || [];
    });
    
    if (errors.length > 0) {
      throw new Error(`Console errors: ${errors.join(', ')}`);
    }

    // Visual regression testing
    await page.screenshot({
      path: `screenshots/${context.id}.png`,
      fullPage: true
    });
  },
};
```

### Component Visual Tests

```javascript
// shared/src/components/__tests__/Button.visual.test.js
import { test, expect } from '@playwright/test';

test.describe('Button Visual Tests', () => {
  test('renders all button variants correctly', async ({ page }) => {
    await page.goto('/storybook/button');
    
    // Test different variants
    const variants = ['primary', 'secondary', 'danger', 'outline'];
    
    for (const variant of variants) {
      await page.click(`[data-variant="${variant}"]`);
      await expect(page).toHaveScreenshot(`button-${variant}.png`);
    }
  });

  test('button states visual consistency', async ({ page }) => {
    await page.goto('/storybook/button');
    
    // Normal state
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-normal.png');
    
    // Hover state
    await page.hover('[data-testid="button"]');
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-hover.png');
    
    // Disabled state
    await page.click('[data-testid="toggle-disabled"]');
    await expect(page.locator('[data-testid="button"]')).toHaveScreenshot('button-disabled.png');
  });
});
```

## 🔍 Testing Microservice Boundaries

### Boundary Testing

```javascript
// shell/src/__tests__/serviceBoundaries.test.js
describe('Service Boundaries', () => {
  it('services maintain independent state', async () => {
    const { UserService } = await import('user_service/UserApp');
    const { ProductService } = await import('product_service/ProductApp');
    
    // Mount both services
    const userContainer = document.createElement('div');
    const productContainer = document.createElement('div');
    
    await UserService.mount(userContainer, { userId: '123' });
    await ProductService.mount(productContainer, { categoryId: '456' });
    
    // Verify isolation
    expect(userContainer.querySelector('[data-service="user"]')).toBeTruthy();
    expect(productContainer.querySelector('[data-service="product"]')).toBeTruthy();
    
    // State changes in one service shouldn't affect another
    const userState = await UserService.getState();
    await ProductService.updateState({ selectedProduct: '789' });
    const userStateAfter = await UserService.getState();
    
    expect(userState).toEqual(userStateAfter);
  });

  it('services communicate only through defined interfaces', () => {
    const eventBus = new MockEventBus();
    
    // Register services with event bus
    const userService = new UserService(eventBus);
    const productService = new ProductService(eventBus);
    
    // Test allowed communication
    const mockHandler = jest.fn();
    productService.on('user:preferences_updated', mockHandler);
    
    userService.updatePreferences({ theme: 'dark' });
    
    expect(mockHandler).toHaveBeenCalledWith({
      userId: expect.any(String),
      preferences: { theme: 'dark' }
    });
  });
});
```

## 📊 Test Reporting and Analytics

### Custom Test Reporter

```javascript
// test-utils/customReporter.js
class MicroserviceTestReporter {
  constructor(globalConfig, options) {
    this.globalConfig = globalConfig;
    this.options = options;
    this.results = {
      services: {},
      integration: {},
      e2e: {}
    };
  }

  onRunStart() {
    console.log('🚀 Starting microservice test suite...');
  }

  onTestResult(test, testResult) {
    const serviceName = this.extractServiceName(test.path);
    
    if (!this.results.services[serviceName]) {
      this.results.services[serviceName] = {
        passed: 0,
        failed: 0,
        coverage: 0
      };
    }

    testResult.testResults.forEach(result => {
      if (result.status === 'passed') {
        this.results.services[serviceName].passed++;
      } else {
        this.results.services[serviceName].failed++;
      }
    });
  }

  onRunComplete() {
    this.generateReport();
  }

  generateReport() {
    const report = {
      timestamp: new Date().toISOString(),
      summary: this.generateSummary(),
      services: this.results.services,
      recommendations: this.generateRecommendations()
    };

    console.table(this.results.services);
    
    // Save detailed report
    require('fs').writeFileSync(
      'test-results/microservice-report.json',
      JSON.stringify(report, null, 2)
    );
  }

  extractServiceName(testPath) {
    const match = testPath.match(/\/([^\/]+)\/src\//);
    return match ? match[1] : 'unknown';
  }

  generateSummary() {
    let totalPassed = 0;
    let totalFailed = 0;

    Object.values(this.results.services).forEach(service => {
      totalPassed += service.passed;
      totalFailed += service.failed;
    });

    return {
      total: totalPassed + totalFailed,
      passed: totalPassed,
      failed: totalFailed,
      success_rate: (totalPassed / (totalPassed + totalFailed)) * 100
    };
  }

  generateRecommendations() {
    const recommendations = [];
    
    Object.entries(this.results.services).forEach(([service, results]) => {
      const failureRate = results.failed / (results.passed + results.failed);
      
      if (failureRate > 0.1) {
        recommendations.push(`🔴 ${service}: High failure rate (${(failureRate * 100).toFixed(1)}%) - needs attention`);
      }
      
      if (results.coverage < 70) {
        recommendations.push(`📊 ${service}: Low coverage (${results.coverage}%) - add more tests`);
      }
    });

    return recommendations;
  }
}

module.exports = MicroserviceTestReporter;
```

## 🚀 CI/CD Testing Pipeline

### GitHub Actions Workflow

```yaml
# .github/workflows/test.yml
name: Test Pipeline

on:
  push:
    branches: [main, develop]
  pull_request:
    branches: [main]

jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
      
      - name: Run unit tests
        run: npm run test:unit -- --coverage
        
      - name: Upload coverage
        uses: codecov/codecov-action@v3

  integration-tests:
    runs-on: ubuntu-latest
    needs: unit-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
        
      - name: Start test services
        run: |
          docker-compose -f docker-compose.test.yml up -d
          npm run wait-for-services
          
      - name: Run integration tests
        run: npm run test:integration
        
      - name: Run contract tests
        run: npm run test:contracts

  e2e-tests:
    runs-on: ubuntu-latest
    needs: integration-tests
    steps:
      - uses: actions/checkout@v3
      
      - name: Setup Node.js
        uses: actions/setup-node@v3
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install dependencies
        run: npm ci
        
      - name: Install Playwright
        run: npx playwright install
        
      - name: Build and start application
        run: |
          npm run build:all
          npm run start:test &
          npm run wait-for-app
          
      - name: Run E2E tests
        run: npm run test:e2e
        
      - name: Upload test results
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: e2e-report
          path: playwright-report/
```

## 📋 Testing Best Practices

### Service-Specific Guidelines

1. **Unit Tests (70% of test suite)**
   - Test business logic in isolation
   - Mock external dependencies
   - Fast execution (< 1ms per test)
   - High code coverage (>80%)

2. **Integration Tests (20% of test suite)**
   - Test service interactions
   - Use real dependencies where possible
   - Focus on data flow and contracts
   - Medium execution time (< 100ms per test)

3. **E2E Tests (10% of test suite)**
   - Test critical user journeys
   - Use production-like environment
   - Focus on high-value scenarios
   - Slower execution acceptable

### Quality Gates

```javascript
// scripts/quality-gates.js
const fs = require('fs');

function checkQualityGates() {
  const coverage = JSON.parse(fs.readFileSync('coverage/coverage-summary.json'));
  const testResults = JSON.parse(fs.readFileSync('test-results.json'));
  
  const gates = {
    coverage: coverage.total.lines.pct >= 80,
    testSuccess: testResults.success >= 95,
    performance: testResults.avgDuration < 5000
  };
  
  const passed = Object.values(gates).every(gate => gate);
  
  if (!passed) {
    console.error('❌ Quality gates failed:', gates);
    process.exit(1);
  }
  
  console.log('✅ All quality gates passed');
}

checkQualityGates();
```
This comprehensive testing strategy ensures reliability, maintainability, and confidence in your frontend microservices architecture. The pyramid approach balances test coverage with execution speed, while the specialized tools handle the unique challenges of distributed frontend systems.

- [Next Tools Ecosystem](10-tools-ecosystem.md)