# 10 — Testing

## Table of Contents

- [Testing Philosophy](#testing-philosophy)
- [The Testing Pyramid](#the-testing-pyramid)
- [Unit Testing](#unit-testing)
- [Integration Testing](#integration-testing)
- [Contract Testing](#contract-testing)
- [End-to-End Testing](#end-to-end-testing)
- [Performance Testing](#performance-testing)
- [Testing in CI/CD](#testing-in-cicd)
- [Test Data Management](#test-data-management)
- [Anti-patterns](#anti-patterns)
- [Summary & Next Steps](#summary--next-steps)

---

## Testing Philosophy

Microservices create a unique testing challenge: each service can be tested in isolation, but the system's correctness emerges from how services interact. A service with 100% unit test coverage can still fail in production if its assumptions about the behaviour of other services are wrong.

The guiding principles:

- **Test behaviour, not implementation** — tests should survive refactoring. If renaming a private function breaks a test, that test is testing the wrong thing.
- **Confidence over coverage** — 70% coverage on meaningful paths beats 95% coverage on trivial getters and setters.
- **Fast feedback** — unit tests run in milliseconds. Integration tests in seconds. E2E tests in minutes. Keep each layer as fast as possible.
- **Test at the right level** — don't use an E2E test to cover something a unit test can verify just as well.
- **Own your test infrastructure** — flaky tests are worse than no tests. An unreliable test suite trains engineers to ignore failures.

---

## The Testing Pyramid

```mermaid
flowchart TD
    E2E["End-to-End tests\nFew, slow, high confidence\nfull system deployed"]
    INT["Integration tests\nModerate count, seconds\nreal DB, real broker"]
    CON["Contract tests\nOne per API boundary\nconsumer-driven"]
    UNIT["Unit tests\nMany, milliseconds\npure functions, domain logic"]

    UNIT --> CON --> INT --> E2E

    style UNIT fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style CON fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style INT fill:#FAEEDA,stroke:#854F0B,color:#412402
    style E2E fill:#FCEBEB,stroke:#A32D2D,color:#501313
```

| Layer | Count | Speed | Scope | Cost of failure |
|-------|-------|-------|-------|----------------|
| Unit | Hundreds | < 1ms each | Single function / class | Cheap — fix immediately |
| Contract | One per boundary | < 1s | API shape agreement | Medium — coordination needed |
| Integration | Tens | 1–10s | Service + real dependencies | Medium — environment setup |
| E2E | Few (< 20) | 30s–5min | Full user journey | Expensive — slow to debug |

The pyramid shape is intentional: most confidence comes cheaply from unit tests at the base. E2E tests are expensive to write, slow to run, and brittle — use them sparingly for the most critical user journeys only.

---

## Unit Testing

Unit tests verify isolated pieces of logic — a function, a class, a domain rule — without touching the network, database, or filesystem. They are the fastest feedback loop available.

### What to Unit Test

- Domain logic and business rules
- Data transformation and mapping functions
- Validation logic
- Error handling and edge cases
- Algorithm correctness

### What NOT to Unit Test

- Framework wiring (Express route registration)
- Database query syntax (test that with integration tests against a real DB)
- Third-party library internals

### Structure — Arrange / Act / Assert

```typescript
// order.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { OrderService } from './order.service';
import { OrderRepository } from './order.repository';
import { PaymentGateway } from '../payment/payment.gateway';

// Mock dependencies — unit tests do not touch real infrastructure
vi.mock('./order.repository');
vi.mock('../payment/payment.gateway');

describe('OrderService', () => {
  let orderService:    OrderService;
  let orderRepository: vi.Mocked<OrderRepository>;
  let paymentGateway:  vi.Mocked<PaymentGateway>;

  beforeEach(() => {
    orderRepository = new OrderRepository() as vi.Mocked<OrderRepository>;
    paymentGateway  = new PaymentGateway()  as vi.Mocked<PaymentGateway>;
    orderService    = new OrderService(orderRepository, paymentGateway);
  });

  describe('createOrder', () => {
    it('creates an order and returns it with PENDING status', async () => {
      // Arrange
      const input = {
        userId:  'usr_123',
        items:   [{ productId: 'prod_456', quantity: 2, unitPrice: 49.99 }],
        currency: 'USD',
      };
      const expectedOrder = { id: 'ord_789', ...input, status: 'PENDING', total: 99.98 };

      orderRepository.create.mockResolvedValue(expectedOrder);

      // Act
      const result = await orderService.createOrder(input);

      // Assert
      expect(result).toEqual(expectedOrder);
      expect(result.status).toBe('PENDING');
      expect(orderRepository.create).toHaveBeenCalledWith(
        expect.objectContaining({ userId: 'usr_123' }),
      );
    });

    it('throws ValidationError when items array is empty', async () => {
      const input = { userId: 'usr_123', items: [], currency: 'USD' };

      await expect(orderService.createOrder(input))
        .rejects
        .toThrow('Order must contain at least one item');

      expect(orderRepository.create).not.toHaveBeenCalled();
    });

    it('calculates total correctly with multiple items', async () => {
      const input = {
        userId:  'usr_123',
        currency: 'USD',
        items:   [
          { productId: 'prod_1', quantity: 2, unitPrice: 10.00 },
          { productId: 'prod_2', quantity: 1, unitPrice: 5.50 },
        ],
      };
      orderRepository.create.mockResolvedValue({ id: 'ord_1', ...input, status: 'PENDING', total: 25.50 });

      const result = await orderService.createOrder(input);

      expect(result.total).toBe(25.50);
    });

    it('does not call payment gateway during order creation', async () => {
      const input = { userId: 'usr_123', items: [{ productId: 'prod_1', quantity: 1, unitPrice: 10 }], currency: 'USD' };
      orderRepository.create.mockResolvedValue({ id: 'ord_1', ...input, status: 'PENDING', total: 10 });

      await orderService.createOrder(input);

      expect(paymentGateway.charge).not.toHaveBeenCalled();
    });
  });

  describe('cancelOrder', () => {
    it('throws NotFoundError when order does not exist', async () => {
      orderRepository.findById.mockResolvedValue(null);

      await expect(orderService.cancelOrder('ord_nonexistent', 'usr_123'))
        .rejects
        .toThrow('Order not found');
    });

    it('throws ForbiddenError when user does not own the order', async () => {
      orderRepository.findById.mockResolvedValue({
        id: 'ord_789', userId: 'usr_OTHER', status: 'PENDING',
      });

      await expect(orderService.cancelOrder('ord_789', 'usr_123'))
        .rejects
        .toThrow('Forbidden');
    });

    it('throws ConflictError when order is already shipped', async () => {
      orderRepository.findById.mockResolvedValue({
        id: 'ord_789', userId: 'usr_123', status: 'SHIPPED',
      });

      await expect(orderService.cancelOrder('ord_789', 'usr_123'))
        .rejects
        .toThrow('Cannot cancel a shipped order');
    });
  });
});
```

### Testing Domain Entities

Pure domain logic with no dependencies is the easiest to test — no mocking required.

```typescript
// order.entity.test.ts
import { Order } from './order.entity';

describe('Order', () => {
  it('can be cancelled when PENDING', () => {
    const order = Order.create({ userId: 'usr_1', items: [...] });
    order.cancel();
    expect(order.status).toBe('CANCELLED');
  });

  it('cannot be cancelled when already SHIPPED', () => {
    const order = Order.reconstitute({ status: 'SHIPPED', ... });
    expect(() => order.cancel()).toThrow('Cannot cancel a shipped order');
  });

  it('emits OrderCancelled domain event on cancel', () => {
    const order = Order.create({ userId: 'usr_1', items: [...] });
    order.cancel();
    const events = order.pullDomainEvents();
    expect(events).toContainEqual(
      expect.objectContaining({ type: 'OrderCancelled', orderId: order.id }),
    );
  });
});
```

---

## Integration Testing

Integration tests verify that your service works correctly with real infrastructure — a real database, a real message broker, a real cache. They catch the class of bugs that unit tests cannot: incorrect SQL, wrong Redis key format, misconfigured transactions.

### Test Containers — Real Infrastructure, No Mocks

Testcontainers spins up Docker containers for the duration of the test suite. No shared test environments, no flaky shared databases.

```typescript
// order.repository.integration.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { PostgreSqlContainer, StartedPostgreSqlContainer } from '@testcontainers/postgresql';
import { Pool } from 'pg';
import { OrderRepository } from './order.repository';
import { runMigrations } from '../db/migrations';

describe('OrderRepository (integration)', () => {
  let container:       StartedPostgreSqlContainer;
  let pool:            Pool;
  let orderRepository: OrderRepository;

  beforeAll(async () => {
    // Start a real PostgreSQL container
    container = await new PostgreSqlContainer('postgres:16-alpine')
      .withDatabase('orders_test')
      .withUsername('test')
      .withPassword('test')
      .start();

    pool = new Pool({ connectionString: container.getConnectionUri() });

    // Run migrations against the test database
    await runMigrations(pool);

    orderRepository = new OrderRepository(pool);
  }, 60_000); // containers can take time to start

  afterAll(async () => {
    await pool.end();
    await container.stop();
  });

  beforeEach(async () => {
    // Clean slate for each test — truncate all tables
    await pool.query('TRUNCATE orders, order_items CASCADE');
  });

  it('creates an order and retrieves it by ID', async () => {
    const created = await orderRepository.create({
      userId:   'usr_test',
      currency: 'USD',
      items:    [{ productId: 'prod_1', quantity: 2, unitPrice: 49.99 }],
    });

    expect(created.id).toBeDefined();
    expect(created.status).toBe('PENDING');

    const found = await orderRepository.findById(created.id);
    expect(found).toMatchObject({
      id:       created.id,
      userId:   'usr_test',
      currency: 'USD',
    });
  });

  it('returns null when order does not exist', async () => {
    const result = await orderRepository.findById('ord_nonexistent');
    expect(result).toBeNull();
  });

  it('finds all orders for a user ordered by creation date descending', async () => {
    await orderRepository.create({ userId: 'usr_1', currency: 'USD', items: [...] });
    await sleep(10);
    await orderRepository.create({ userId: 'usr_1', currency: 'USD', items: [...] });
    await orderRepository.create({ userId: 'usr_2', currency: 'USD', items: [...] }); // different user

    const orders = await orderRepository.findByUserId('usr_1');

    expect(orders).toHaveLength(2);
    expect(orders[0].createdAt.getTime()).toBeGreaterThan(orders[1].createdAt.getTime());
  });

  it('updates order status atomically', async () => {
    const order = await orderRepository.create({ userId: 'usr_1', currency: 'USD', items: [...] });

    await orderRepository.updateStatus(order.id, 'CONFIRMED');

    const updated = await orderRepository.findById(order.id);
    expect(updated?.status).toBe('CONFIRMED');
  });
});
```

### HTTP Integration Tests — Testing the Full HTTP Layer

```typescript
// order.routes.integration.test.ts
import request from 'supertest';
import { app } from '../app';

describe('POST /api/v1/orders', () => {
  it('returns 201 with the created order', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${testJwt}`)
      .send({
        items:    [{ productId: 'prod_123', quantity: 1 }],
        currency: 'USD',
      });

    expect(response.status).toBe(201);
    expect(response.body).toMatchObject({
      id:       expect.stringMatching(/^ord_/),
      status:   'PENDING',
      currency: 'USD',
    });
    expect(response.headers['location']).toBe(`/api/v1/orders/${response.body.id}`);
  });

  it('returns 422 when items is empty', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${testJwt}`)
      .send({ items: [], currency: 'USD' });

    expect(response.status).toBe(422);
    expect(response.body.error.code).toBe('VALIDATION_ERROR');
    expect(response.body.error.details).toContainEqual(
      expect.objectContaining({ field: 'items' }),
    );
  });

  it('returns 401 when no token is provided', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .send({ items: [{ productId: 'prod_123', quantity: 1 }] });

    expect(response.status).toBe(401);
  });

  it('returns 429 when rate limit is exceeded', async () => {
    // Fire 61 requests — limit is 60/min
    const requests = Array.from({ length: 61 }, () =>
      request(app)
        .post('/api/v1/orders')
        .set('Authorization', `Bearer ${testJwt}`)
        .send({ items: [{ productId: 'prod_123', quantity: 1 }] }),
    );

    const responses = await Promise.all(requests);
    const tooMany = responses.filter(r => r.status === 429);
    expect(tooMany.length).toBeGreaterThanOrEqual(1);
  });
});
```

---

## Contract Testing

Contract testing is the most underused and most important test type in microservices. It answers: *does this service still honour the API contract that its consumers depend on?*

Without contract tests, the only way to verify cross-service compatibility is with expensive E2E tests or with manual testing before every release.

### Consumer-Driven Contract Testing with Pact

```mermaid
flowchart LR
    C[Consumer\nOrder Service] -->|1. writes expected interactions| P[Pact Broker]
    P -->|2. publishes contract| PR[Provider\nUser Service]
    PR -->|3. verifies contract| P
    P -->|4. can-i-deploy check| CI[CI Pipeline]

    style C fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style P fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style PR fill:#FAEEDA,stroke:#854F0B,color:#412402
    style CI fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
```

**Step 1 — Consumer writes the contract (Order Service tests)**

```typescript
// order-service/src/user-client.pact.test.ts
import { PactV3, MatchersV3 } from '@pact-foundation/pact';
import { UserClient } from './user.client';

const { like, string, uuid } = MatchersV3;

const provider = new PactV3({
  consumer: 'order-service',
  provider: 'user-service',
  dir:      './pacts',
});

describe('UserClient — Pact contract', () => {
  it('fetches a user by ID', async () => {
    await provider
      .given('a user with ID usr_123 exists')
      .uponReceiving('a request to get user usr_123')
      .withRequest({
        method: 'GET',
        path:   '/api/v1/users/usr_123',
        headers: { Authorization: like('Bearer some-token') },
      })
      .willRespondWith({
        status: 200,
        headers: { 'Content-Type': 'application/json' },
        body: {
          id:          uuid(),
          email:       string('alice@example.com'),
          displayName: string('Alice'),
          createdAt:   string('2024-01-01T00:00:00Z'),
        },
      })
      .executeTest(async (mockServer) => {
        const client = new UserClient(mockServer.url);
        const user   = await client.getById('usr_123');

        expect(user.id).toBeDefined();
        expect(user.email).toBeDefined();
        expect(user.displayName).toBeDefined();
      });
  });

  it('returns 404 when user does not exist', async () => {
    await provider
      .given('no user with ID usr_notfound exists')
      .uponReceiving('a request for a non-existent user')
      .withRequest({
        method:  'GET',
        path:    '/api/v1/users/usr_notfound',
        headers: { Authorization: like('Bearer some-token') },
      })
      .willRespondWith({
        status: 404,
        body:   { error: { code: string('NOT_FOUND') } },
      })
      .executeTest(async (mockServer) => {
        const client = new UserClient(mockServer.url);

        await expect(client.getById('usr_notfound'))
          .rejects.toThrow('User not found');
      });
  });
});
```

**Step 2 — Provider verifies the contract (User Service tests)**

```typescript
// user-service/src/pact.verification.test.ts
import { Verifier } from '@pact-foundation/pact';
import { app } from '../app';
import { seedUser } from '../test/fixtures';

describe('Pact provider verification', () => {
  it('satisfies all consumer contracts', async () => {
    const server = app.listen(0); // random port
    const port   = (server.address() as AddressInfo).port;

    await new Verifier({
      provider:        'user-service',
      providerBaseUrl: `http://localhost:${port}`,

      // Pull contracts from the Pact Broker
      pactBrokerUrl:         process.env.PACT_BROKER_URL,
      pactBrokerToken:       process.env.PACT_BROKER_TOKEN,
      publishVerificationResults: process.env.CI === 'true',
      providerVersion:       process.env.GIT_SHA,

      // Provider states — set up test data for each state
      stateHandlers: {
        'a user with ID usr_123 exists': async () => {
          await seedUser({ id: 'usr_123', email: 'alice@example.com', displayName: 'Alice' });
        },
        'no user with ID usr_notfound exists': async () => {
          // nothing — user doesn't exist
        },
      },
    }).verifyProvider();

    server.close();
  });
});
```

**Step 3 — Gate deployments with `can-i-deploy`**

```yaml
# In CI pipeline — before deploying user-service to production
- name: Check if safe to deploy
  run: |
    npx pact-broker can-i-deploy \
      --pacticipant user-service \
      --version ${GIT_SHA} \
      --to-environment production \
      --broker-base-url ${{ secrets.PACT_BROKER_URL }} \
      --broker-token   ${{ secrets.PACT_BROKER_TOKEN }}
```

If any consumer's contract is not yet verified against this version, `can-i-deploy` fails and the deployment is blocked. This is the safety net that makes independent deployments safe.

---

## End-to-End Testing

E2E tests deploy the full system (or a representative subset) and drive it through real user journeys. They are the most expensive tests to write, maintain, and run — use them sparingly.

### What E2E Tests Are For

- Critical happy paths that must never break (place order, make payment, receive confirmation)
- Journeys that cross three or more services in a single flow
- Smoke tests run immediately after a production deployment

### What E2E Tests Are NOT For

- Covering every edge case and error condition (that's unit/integration territory)
- Validating UI behaviour (use component tests instead)
- Replacing contract tests as a compatibility check

### Example E2E Test (Supertest Against Deployed Staging)

```typescript
// tests/e2e/order-journey.test.ts
import request from 'supertest';

const STAGING_BASE_URL = process.env.STAGING_URL ?? 'https://staging.example.com';

describe('Order journey — E2E', () => {
  let authToken:  string;
  let testUserId: string;

  beforeAll(async () => {
    // Authenticate a test user
    const authRes = await request(STAGING_BASE_URL)
      .post('/api/v1/auth/token')
      .send({ email: 'e2e-test@example.com', password: process.env.E2E_TEST_PASSWORD });

    authToken  = authRes.body.accessToken;
    testUserId = authRes.body.userId;
  });

  it('completes a full order placement journey', async () => {
    // 1. Place an order
    const orderRes = await request(STAGING_BASE_URL)
      .post('/api/v1/orders')
      .set('Authorization', `Bearer ${authToken}`)
      .send({
        items:            [{ productId: 'prod_e2e_test', quantity: 1 }],
        paymentMethodId:  'pm_test_card',
        shippingAddressId: 'addr_test',
      });

    expect(orderRes.status).toBe(201);
    const orderId = orderRes.body.id;

    // 2. Verify order is retrievable
    const getOrderRes = await request(STAGING_BASE_URL)
      .get(`/api/v1/orders/${orderId}`)
      .set('Authorization', `Bearer ${authToken}`);

    expect(getOrderRes.status).toBe(200);
    expect(getOrderRes.body.status).toBe('PENDING');

    // 3. Wait for payment processing (async) — poll with timeout
    const confirmedOrder = await pollUntil(
      () =>
        request(STAGING_BASE_URL)
          .get(`/api/v1/orders/${orderId}`)
          .set('Authorization', `Bearer ${authToken}`)
          .then(r => r.body),
      (order) => order.status === 'CONFIRMED',
      { intervalMs: 1000, timeoutMs: 15_000 },
    );

    expect(confirmedOrder.status).toBe('CONFIRMED');
    expect(confirmedOrder.paymentId).toBeDefined();
  });

  afterAll(async () => {
    // Clean up test data — delete the test user's orders
    await cleanupTestOrders(testUserId, authToken);
  });
});

async function pollUntil<T>(
  fn:      () => Promise<T>,
  until:   (val: T) => boolean,
  options: { intervalMs: number; timeoutMs: number },
): Promise<T> {
  const deadline = Date.now() + options.timeoutMs;
  while (Date.now() < deadline) {
    const result = await fn();
    if (until(result)) return result;
    await sleep(options.intervalMs);
  }
  throw new Error(`pollUntil timed out after ${options.timeoutMs}ms`);
}
```

---

## Performance Testing

Performance tests verify that the service meets its latency and throughput SLOs under realistic load. Run them against staging before every significant release — and on a schedule to catch performance regressions.

### Load Test Scenarios

```mermaid
flowchart LR
    BL[Baseline\nsteady state\nverify P99 < SLO] --> ST[Stress test\nramp beyond capacity\nfind breaking point]
    ST --> SP[Spike test\ninstant surge\ntest autoscaling]
    SP --> SO[Soak test\nextended run\nfind memory leaks]

    style BL fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style ST fill:#FAEEDA,stroke:#854F0B,color:#412402
    style SP fill:#FCEBEB,stroke:#A32D2D,color:#501313
    style SO fill:#EEEDFE,stroke:#534AB7,color:#26215C
```

See [09-performance-optimization.md](./09-performance-optimization.md) for full k6 load test implementation. Performance test thresholds should directly reflect your SLOs from [07-observability.md](./07-observability.md).

```javascript
// performance/order-service.perf.test.js (k6)
export const options = {
  scenarios: {
    baseline: {
      executor:        'constant-vus',
      vus:             50,
      duration:        '5m',
    },
    stress: {
      executor:        'ramping-vus',
      startVUs:        0,
      stages:          [
        { duration: '2m', target: 200 },
        { duration: '5m', target: 200 },
        { duration: '2m', target: 0   },
      ],
      startTime:       '6m', // run after baseline
    },
  },
  thresholds: {
    'http_req_duration{scenario:baseline}': ['p(99)<500'],  // SLO: P99 < 500ms
    'http_req_failed{scenario:baseline}':   ['rate<0.001'], // SLO: < 0.1% errors
    'http_req_duration{scenario:stress}':   ['p(99)<1500'], // degraded but not broken
  },
};
```

---

## Testing in CI/CD

Every pipeline stage has a corresponding test suite. Tests are the gates that prevent bad code from reaching production.

```mermaid
flowchart LR
    Push[git push] --> U[Unit tests\n< 30s]
    U --> I[Integration tests\n< 3 min]
    I --> C[Contract tests\n< 1 min]
    C --> Build[Build image]
    Build --> Staging[Deploy staging]
    Staging --> E2E[E2E smoke tests\n< 5 min]
    E2E --> Perf[Performance tests\nweekly / on demand]
    Perf --> Prod[Deploy production]

    style Push fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style U fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style I fill:#E1F5EE,stroke:#0F6E56,color:#04342C
    style C fill:#EEEDFE,stroke:#534AB7,color:#26215C
    style Build fill:#F1EFE8,stroke:#5F5E5A,color:#2C2C2A
    style Staging fill:#FAEEDA,stroke:#854F0B,color:#412402
    style E2E fill:#FAEEDA,stroke:#854F0B,color:#412402
    style Perf fill:#FCEBEB,stroke:#A32D2D,color:#501313
    style Prod fill:#E1F5EE,stroke:#0F6E56,color:#04342C
```

### Parallel Test Execution

```yaml
# .github/workflows/test.yml
jobs:
  unit-tests:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:unit -- --coverage
      - uses: actions/upload-artifact@v4
        with:
          name: coverage-report
          path: coverage/

  integration-tests:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:16-alpine
        env:
          POSTGRES_DB:       orders_test
          POSTGRES_USER:     test
          POSTGRES_PASSWORD: test
        ports: ['5432:5432']
        options: --health-cmd pg_isready --health-interval 5s --health-timeout 5s
      redis:
        image: redis:7-alpine
        ports: ['6379:6379']
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run db:migrate:test
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/orders_test
      - run: npm run test:integration
        env:
          DATABASE_URL: postgresql://test:test@localhost:5432/orders_test
          REDIS_URL:    redis://localhost:6379

  contract-tests:
    runs-on: ubuntu-latest
    needs: [unit-tests]   # run after unit tests pass
    steps:
      - uses: actions/checkout@v4
      - run: npm ci
      - run: npm run test:pact
        env:
          PACT_BROKER_URL:   ${{ secrets.PACT_BROKER_URL }}
          PACT_BROKER_TOKEN: ${{ secrets.PACT_BROKER_TOKEN }}
          GIT_SHA:           ${{ github.sha }}

  # unit, integration, and contract tests run in parallel
  build:
    needs: [unit-tests, integration-tests, contract-tests]
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - run: docker build -t order-service:${{ github.sha }} .
```

### Test Coverage Enforcement

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    coverage: {
      provider:       'v8',
      reporter:       ['text', 'lcov', 'html'],
      thresholds: {
        lines:      80,
        functions:  80,
        branches:   75,
        statements: 80,
      },
      // Don't count infrastructure wiring and generated code in coverage
      exclude: [
        'src/infrastructure/**',
        'src/**/*.dto.ts',
        'src/**/*.config.ts',
        'src/**/index.ts',
      ],
    },
  },
});
```

---

## Test Data Management

### Fixtures and Factories

```typescript
// test/factories/order.factory.ts
import { faker } from '@faker-js/faker';
import type { Order, OrderItem } from '../../src/domain/order.entity';

export function buildOrderItem(overrides: Partial<OrderItem> = {}): OrderItem {
  return {
    productId: faker.string.uuid(),
    quantity:  faker.number.int({ min: 1, max: 10 }),
    unitPrice: faker.number.float({ min: 1, max: 999, fractionDigits: 2 }),
    ...overrides,
  };
}

export function buildOrder(overrides: Partial<Order> = {}): Order {
  return {
    id:        `ord_${faker.string.alphanumeric(8)}`,
    userId:    `usr_${faker.string.alphanumeric(8)}`,
    status:    'PENDING',
    currency:  'USD',
    items:     [buildOrderItem()],
    total:     faker.number.float({ min: 1, max: 9999, fractionDigits: 2 }),
    createdAt: faker.date.recent(),
    updatedAt: faker.date.recent(),
    ...overrides,
  };
}

// Convenience builders for common scenarios
export const aConfirmedOrder = () => buildOrder({ status: 'CONFIRMED' });
export const aShippedOrder   = () => buildOrder({ status: 'SHIPPED'   });
export const aCancelledOrder = () => buildOrder({ status: 'CANCELLED' });
```

### Database Seeding for Integration Tests

```typescript
// test/fixtures/seed.ts
import { Pool } from 'pg';
import { buildOrder, buildOrderItem } from '../factories/order.factory';

export async function seedOrder(
  pool:     Pool,
  overrides: Partial<Order> = {},
): Promise<Order> {
  const order = buildOrder(overrides);

  await pool.query(
    `INSERT INTO orders (id, user_id, status, currency, total, created_at, updated_at)
     VALUES ($1, $2, $3, $4, $5, $6, $7)`,
    [order.id, order.userId, order.status, order.currency, order.total, order.createdAt, order.updatedAt],
  );

  for (const item of order.items) {
    await pool.query(
      `INSERT INTO order_items (order_id, product_id, quantity, unit_price)
       VALUES ($1, $2, $3, $4)`,
      [order.id, item.productId, item.quantity, item.unitPrice],
    );
  }

  return order;
}

export async function truncateAll(pool: Pool): Promise<void> {
  await pool.query('TRUNCATE orders, order_items CASCADE');
}
```

### Test Isolation Strategies

| Strategy | How | When |
|----------|-----|------|
| Truncate between tests | `TRUNCATE table CASCADE` in `beforeEach` | Default for integration tests |
| Transaction rollback | Wrap each test in a transaction, rollback at end | Faster than truncate, needs careful handling with nested transactions |
| Schema per test | Each test gets its own schema | Slowest, but fully isolated — use for parallel test suites |
| Seed-then-read | Seed specific data, assert against it | Cleaner than truncate — know exactly what's in the DB |

---

## Anti-patterns

### Testing Implementation Details

```typescript
// BAD — test breaks if you rename the private method
it('calls _validateItems internally', () => {
  const spy = vi.spyOn(orderService as any, '_validateItems');
  await orderService.createOrder(input);
  expect(spy).toHaveBeenCalled(); // tests HOW, not WHAT
});

// GOOD — test the observable outcome
it('rejects orders with no items', async () => {
  await expect(orderService.createOrder({ ...input, items: [] }))
    .rejects.toThrow('Order must contain at least one item');
});
```

### Mocking the Thing You Are Testing

```typescript
// BAD — mocking the database inside an integration test defeats the point
vi.mock('../db/pool', () => ({ query: vi.fn().mockResolvedValue({ rows: [] }) }));

// Integration tests must use a real database (Testcontainers)
```

### Brittle Assertions

```typescript
// BAD — breaks when any unrelated field changes
expect(order).toEqual({
  id:        'ord_123',
  userId:    'usr_456',
  status:    'PENDING',
  total:     99.99,
  currency:  'USD',
  createdAt: '2024-01-15T10:30:00.000Z', // will fail a millisecond later
  updatedAt: '2024-01-15T10:30:00.000Z',
  internalField: 'value',                // leaking internals
});

// GOOD — assert what you care about
expect(order).toMatchObject({
  userId:   'usr_456',
  status:   'PENDING',
  currency: 'USD',
});
expect(order.id).toMatch(/^ord_/);
expect(order.createdAt).toBeInstanceOf(Date);
```

### Shared Mutable Test State

```typescript
// BAD — tests interfere with each other through shared state
let order: Order; // shared across tests

it('creates an order', async () => {
  order = await orderService.createOrder(input); // sets shared state
});

it('cancels the order', async () => {
  await orderService.cancelOrder(order.id); // depends on previous test running first
});

// GOOD — each test creates its own data
it('cancels a pending order', async () => {
  const order = await orderService.createOrder(input); // local to this test
  await orderService.cancelOrder(order.id);
  expect(await orderService.getById(order.id)).toMatchObject({ status: 'CANCELLED' });
});
```

### Flaky Async Tests

```typescript
// BAD — race condition: the event may not have arrived yet
await orderService.createOrder(input);
expect(emailSpy).toHaveBeenCalledOnce(); // may fail if email is async

// GOOD — wait for the side effect explicitly
await orderService.createOrder(input);
await vi.waitFor(() => expect(emailSpy).toHaveBeenCalledOnce(), { timeout: 5000 });

// Or — test async behaviour via the queue, not the spy
const jobs = await emailQueue.getJobs(['waiting', 'active', 'completed']);
expect(jobs).toHaveLength(1);
expect(jobs[0].data.orderId).toBe(order.id);
```

---

## Summary & Next Steps

Testing in microservices is not just about covering lines of code — it is about building justified confidence that services behave correctly in isolation and that they can evolve independently without breaking their neighbours.

The priorities in order:

1. **Unit tests** — write them for all domain logic. Fast, focused, no infrastructure.
2. **Contract tests** — one per API boundary. The safety net that makes independent deployments safe.
3. **Integration tests** — test against real databases and brokers using Testcontainers. Catch what unit tests miss.
4. **E2E tests** — ten to twenty critical journeys only. Run after every staging deployment.
5. **Performance tests** — run on a schedule and before major releases. Gate on SLO thresholds.

The most common failure mode is skipping contract tests entirely and relying on E2E tests for compatibility verification. This is slow, brittle, and does not scale beyond five or six services. Add Pact early.

### Recommended Reading Order

| Step | Document | What you'll learn |
|------|----------|------------------|
| Next | [11-tools-ecosystem.md](./11-tools-ecosystem.md) | Service mesh, API gateways, and observability tooling |
| Also | [07-observability.md](./07-observability.md) | SLOs that your performance tests should validate |
| Also | [05-deployment-strategies.md](./05-deployment-strategies.md) | Where tests fit in the CI/CD pipeline |

---

*Part of the [Microservices Architecture Guide](../../README.md)*  
*Previous: [09-performance-optimization.md](./09-performance-optimization.md)*  
*Next: [11-tools-ecosystem.md](./11-tools-ecosystem.md)*