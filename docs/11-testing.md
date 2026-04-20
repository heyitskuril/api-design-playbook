# 11. Testing APIs

## What to Test and What Not To Test

API testing lives in the integration test layer. You are testing that the HTTP interface, validation, service logic, and database work together correctly — not that individual functions return the right value (unit tests) and not that a full user journey works in a browser (E2E tests).

For an Express API, this means making real HTTP requests against the application, against a real test database, and asserting on HTTP status codes, response bodies, and database state.

What you should test:
- Every endpoint's happy path — the success case with valid input
- Every validation failure case — missing fields, wrong types, constraint violations
- Authorization cases — anonymous request to a protected endpoint, insufficient role
- Business logic failure cases — not found, conflict, account not verified
- Side effects — that a POST actually creates a record, that a DELETE actually removes it

What you do not need to test at the integration level:
- Individual service methods in isolation — that is unit testing territory
- UI behavior — not an API concern
- Every possible combination of filter parameters — test representative cases

---

## Test Setup

### Configuration

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
    globalSetup: ['./src/test/global-setup.ts'],
    testTimeout: 10_000, // 10 seconds — allow for DB operations
    coverage: {
      provider: 'v8',
      reporter: ['text', 'html'],
      include: ['src/features/**/*.ts'],
      exclude: ['src/features/**/*.router.ts', 'src/features/**/*.types.ts'],
      thresholds: { lines: 80, functions: 80 },
    },
  },
});
```

```typescript
// src/test/global-setup.ts
// Runs once before all tests — set up test database
import { execSync } from 'child_process';

export async function setup() {
  // Apply migrations to the test database
  execSync('npx prisma migrate deploy', {
    env: { ...process.env, DATABASE_URL: process.env.TEST_DATABASE_URL },
  });
}

export async function teardown() {
  // Optional: clean up after all tests
}
```

```typescript
// src/test/setup.ts
// Runs before each test file
import { prisma } from '../infrastructure/database/prisma';
import { beforeEach, afterAll } from 'vitest';

beforeEach(async () => {
  // Clean database between test files — order matters due to foreign keys
  await prisma.$transaction([
    prisma.orderItem.deleteMany(),
    prisma.order.deleteMany(),
    prisma.user.deleteMany(),
  ]);
});

afterAll(async () => {
  await prisma.$disconnect();
});
```

### Test Helpers

Reduce repetition with helper functions for common operations:

```typescript
// src/test/helpers.ts
import request from 'supertest';
import { createApp } from '../app';
import { prisma } from '../infrastructure/database/prisma';
import bcrypt from 'bcrypt';

export const app = createApp();

// Create a test user and return authentication cookies
export async function createAuthenticatedUser(
  overrides: Partial<{ email: string; role: string }> = {},
) {
  const user = await prisma.user.create({
    data: {
      email: overrides.email ?? `test-${Date.now()}@example.com`,
      name: 'Test User',
      passwordHash: await bcrypt.hash('password123', 4), // Low cost factor for tests
      role: (overrides.role as 'member' | 'admin') ?? 'member',
      isVerified: true,
    },
  });

  const response = await request(app)
    .post('/api/v1/auth/login')
    .send({ email: user.email, password: 'password123' });

  const cookies = response.headers['set-cookie'] as string[];

  return { user, cookies };
}

export async function createTestOrder(
  userId: string,
  overrides: Partial<{ status: string }> = {},
) {
  return prisma.order.create({
    data: {
      userId,
      status: (overrides.status as any) ?? 'pending',
      total: 150000,
      items: {
        create: [{ productId: 'test-product-id', quantity: 2, price: 75000 }],
      },
    },
    include: { items: true },
  });
}
```

---

## Writing Integration Tests

### Orders Endpoint

```typescript
// features/orders/orders.router.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app, createAuthenticatedUser, createTestOrder } from '../../test/helpers';
import { prisma } from '../../infrastructure/database/prisma';

describe('GET /api/v1/orders', () => {
  it('returns 401 when unauthenticated', async () => {
    const response = await request(app).get('/api/v1/orders');

    expect(response.status).toBe(401);
    expect(response.body.code).toBe('UNAUTHORIZED');
  });

  it('returns an empty list when user has no orders', async () => {
    const { cookies } = await createAuthenticatedUser();

    const response = await request(app)
      .get('/api/v1/orders')
      .set('Cookie', cookies);

    expect(response.status).toBe(200);
    expect(response.body.status).toBe('success');
    expect(response.body.data).toHaveLength(0);
    expect(response.body.meta).toMatchObject({ limit: 20, nextCursor: null });
  });

  it('returns only the authenticated user\'s orders', async () => {
    const { user: user1, cookies } = await createAuthenticatedUser();
    const { user: user2 } = await createAuthenticatedUser();

    await createTestOrder(user1.id);
    await createTestOrder(user1.id);
    await createTestOrder(user2.id); // Should not appear

    const response = await request(app)
      .get('/api/v1/orders')
      .set('Cookie', cookies);

    expect(response.status).toBe(200);
    expect(response.body.data).toHaveLength(2);
    response.body.data.forEach((order: { userId: string }) => {
      expect(order.userId).toBe(user1.id);
    });
  });

  it('filters by status when provided', async () => {
    const { user, cookies } = await createAuthenticatedUser();

    await createTestOrder(user.id, { status: 'pending' });
    await createTestOrder(user.id, { status: 'completed' });

    const response = await request(app)
      .get('/api/v1/orders?status=pending')
      .set('Cookie', cookies);

    expect(response.status).toBe(200);
    expect(response.body.data).toHaveLength(1);
    expect(response.body.data[0].status).toBe('pending');
  });
});

describe('POST /api/v1/orders', () => {
  it('returns 401 when unauthenticated', async () => {
    const response = await request(app)
      .post('/api/v1/orders')
      .send({ items: [{ productId: 'abc', quantity: 1 }] });

    expect(response.status).toBe(401);
  });

  it('returns 400 when items array is empty', async () => {
    const { cookies } = await createAuthenticatedUser();

    const response = await request(app)
      .post('/api/v1/orders')
      .set('Cookie', cookies)
      .send({ items: [], shippingAddressId: '550e8400-e29b-41d4-a716-446655440000' });

    expect(response.status).toBe(400);
    expect(response.body.code).toBe('VALIDATION_ERROR');
    expect(response.body.details).toEqual(
      expect.arrayContaining([
        expect.objectContaining({ field: 'body.items' }),
      ]),
    );
  });

  it('creates an order and returns 201 with Location header', async () => {
    const { user, cookies } = await createAuthenticatedUser();
    const addressId = '550e8400-e29b-41d4-a716-446655440000';

    // Set up required data
    await prisma.shippingAddress.create({
      data: { id: addressId, userId: user.id, line1: '123 Test St', city: 'Jakarta' },
    });

    const response = await request(app)
      .post('/api/v1/orders')
      .set('Cookie', cookies)
      .send({
        items: [{ productId: 'product-id', quantity: 2 }],
        shippingAddressId: addressId,
      });

    expect(response.status).toBe(201);
    expect(response.body.status).toBe('success');
    expect(response.body.data.id).toBeDefined();
    expect(response.body.data.status).toBe('pending');
    expect(response.headers.location).toContain('/api/v1/orders/');

    // Verify the order was actually created in the database
    const created = await prisma.order.findUnique({ where: { id: response.body.data.id } });
    expect(created).not.toBeNull();
  });

  it('is idempotent when Idempotency-Key header is provided', async () => {
    const { cookies } = await createAuthenticatedUser();
    const idempotencyKey = '550e8400-e29b-41d4-a716-446655440001';

    const payload = {
      items: [{ productId: 'product-id', quantity: 1 }],
      shippingAddressId: '550e8400-e29b-41d4-a716-446655440000',
    };

    const first = await request(app)
      .post('/api/v1/orders')
      .set('Cookie', cookies)
      .set('Idempotency-Key', idempotencyKey)
      .send(payload);

    const second = await request(app)
      .post('/api/v1/orders')
      .set('Cookie', cookies)
      .set('Idempotency-Key', idempotencyKey)
      .send(payload);

    expect(first.status).toBe(201);
    expect(second.status).toBe(201);
    expect(second.headers['idempotent-replayed']).toBe('true');
    expect(second.body.data.id).toBe(first.body.data.id); // Same order returned

    // Only one order created
    const orderCount = await prisma.order.count();
    expect(orderCount).toBe(1);
  });
});

describe('POST /api/v1/orders/:id/cancellation', () => {
  it('cancels a pending order', async () => {
    const { user, cookies } = await createAuthenticatedUser();
    const order = await createTestOrder(user.id, { status: 'pending' });

    const response = await request(app)
      .post(`/api/v1/orders/${order.id}/cancellation`)
      .set('Cookie', cookies);

    expect(response.status).toBe(200);
    expect(response.body.data.status).toBe('cancelled');
  });

  it('returns 409 when order is already cancelled', async () => {
    const { user, cookies } = await createAuthenticatedUser();
    const order = await createTestOrder(user.id, { status: 'cancelled' });

    const response = await request(app)
      .post(`/api/v1/orders/${order.id}/cancellation`)
      .set('Cookie', cookies);

    expect(response.status).toBe(409);
    expect(response.body.code).toBe('ORDER_ALREADY_CANCELLED');
  });

  it('returns 404 when trying to cancel another user\'s order', async () => {
    const { cookies } = await createAuthenticatedUser();
    const { user: anotherUser } = await createAuthenticatedUser();
    const order = await createTestOrder(anotherUser.id);

    const response = await request(app)
      .post(`/api/v1/orders/${order.id}/cancellation`)
      .set('Cookie', cookies);

    // Returns 404 (not 403) to avoid confirming the order exists
    expect(response.status).toBe(404);
  });
});
```

---

## Contract Testing

Integration tests verify your implementation. Contract tests verify your OpenAPI spec matches your implementation. This catches the case where you update the spec but forget to update the code, or vice versa.

```typescript
// src/test/contract.test.ts
import { describe, it, expect } from 'vitest';
import request from 'supertest';
import { app, createAuthenticatedUser } from './helpers';
import Ajv from 'ajv';
import addFormats from 'ajv-formats';
import YAML from 'yamljs';
import path from 'path';

const spec = YAML.load(path.join(__dirname, '../../docs/api/openapi.yaml'));
const ajv = new Ajv({ strict: false });
addFormats(ajv);

describe('Contract: GET /api/v1/orders', () => {
  it('response matches OpenAPI schema', async () => {
    const { cookies } = await createAuthenticatedUser();
    const response = await request(app).get('/api/v1/orders').set('Cookie', cookies);

    const schema = spec.paths['/orders'].get.responses['200'].content['application/json'].schema;
    const validate = ajv.compile(schema);
    const valid = validate(response.body);

    expect(valid).toBe(true);
    if (!valid) {
      console.error('Schema validation errors:', validate.errors);
    }
  });
});
```

---

## Sources

- Khorikov, Vladimir. *Unit Testing: Principles, Practices, and Patterns*. Manning Publications, 2020. — Chapter 8: Why integration testing.
- Supertest Documentation. https://github.com/ladjs/supertest
- Vitest Documentation. https://vitest.dev
- OpenAPI Initiative. "OpenAPI Specification 3.1.0." https://spec.openapis.org/oas/v3.1.0
- AJV Documentation. "JSON Schema Validation." https://ajv.js.org