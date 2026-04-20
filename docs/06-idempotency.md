# 6. Idempotency

## What Idempotency Means and Why It Matters

An operation is idempotent if performing it multiple times produces the same result as performing it once. GET, PUT, and DELETE are idempotent by HTTP specification. POST is not.

The problem idempotency solves is network unreliability. A client sends a request to create a payment. The server receives it, processes the payment, and sends a 201 response. The response is lost in transit — a network timeout, a dropped connection. The client cannot tell whether the request succeeded or failed. If it retries the request, does the payment get charged twice?

Without an idempotency mechanism, yes. With idempotency keys, no.

This matters most for operations that have side effects that must not be duplicated: payments, order creation, email sending, anything that charges money or triggers an irreversible action. Stripe, Adyen, and every serious payment API implements idempotency keys as a first-class feature.

---

## How Idempotency Keys Work

The client generates a unique key (typically a UUID) and includes it in a request header. The server stores the key and the response when processing the request for the first time. On subsequent requests with the same key, the server returns the stored response without re-processing the operation.

```
# First request
POST /api/v1/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "items": [...] }

→ 201 Created
{ "status": "success", "data": { "id": "order-abc", ... } }

# Network drops. Client retries with the same key.
POST /api/v1/orders
Idempotency-Key: 550e8400-e29b-41d4-a716-446655440000
Content-Type: application/json

{ "items": [...] }

→ 201 Created (same response, no duplicate order created)
{ "status": "success", "data": { "id": "order-abc", ... } }
```

The second request returns the same response as the first, but the operation is not repeated.

---

## Implementation

### Database Schema

```prisma
// prisma/schema.prisma
model IdempotencyRecord {
  id             String   @id @default(uuid())
  key            String   @unique
  userId         String
  requestPath    String
  responseStatus Int
  responseBody   Json
  createdAt      DateTime @default(now())
  expiresAt      DateTime

  @@index([key, userId])
  @@index([expiresAt]) // For cleanup job
}
```

### Middleware

```typescript
// shared/middleware/idempotency.ts
import { Request, Response, NextFunction } from 'express';
import { prisma } from '../../infrastructure/database/prisma';
import { AppError } from '../errors/AppError';

const IDEMPOTENCY_TTL_HOURS = 24;

export function idempotency() {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    // Only apply to POST requests — GET, PUT, DELETE are already idempotent
    if (req.method !== 'POST') {
      next();
      return;
    }

    const key = req.headers['idempotency-key'] as string | undefined;

    // If no key provided, proceed normally (idempotency is optional unless you require it)
    if (!key) {
      next();
      return;
    }

    // Validate key format
    const uuidRegex = /^[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
    if (!uuidRegex.test(key)) {
      next(new AppError('INVALID_INPUT', 'Idempotency-Key must be a valid UUID v4.', 400));
      return;
    }

    try {
      const existing = await prisma.idempotencyRecord.findUnique({
        where: { key },
      });

      if (existing) {
        // Validate that this is the same user making the same request
        // Prevents one user from using another user's idempotency key
        if (existing.userId !== req.user?.id) {
          next(new AppError('FORBIDDEN', 'This idempotency key belongs to another user.', 403));
          return;
        }

        if (existing.requestPath !== req.path) {
          next(
            new AppError(
              'CONFLICT',
              'This idempotency key was used for a different endpoint.',
              409,
            ),
          );
          return;
        }

        // Return cached response
        res.setHeader('Idempotent-Replayed', 'true');
        res.status(existing.responseStatus).json(existing.responseBody);
        return;
      }

      // Intercept the response to store it
      const originalJson = res.json.bind(res);
      res.json = (body: unknown) => {
        // Store the response asynchronously — do not block the response
        const expiresAt = new Date();
        expiresAt.setHours(expiresAt.getHours() + IDEMPOTENCY_TTL_HOURS);

        prisma.idempotencyRecord
          .create({
            data: {
              key,
              userId: req.user?.id ?? 'anonymous',
              requestPath: req.path,
              responseStatus: res.statusCode,
              responseBody: body as object,
              expiresAt,
            },
          })
          .catch((err) => {
            // Log but do not fail the request
            console.error('Failed to store idempotency record:', err);
          });

        return originalJson(body);
      };

      next();
    } catch (error) {
      next(error);
    }
  };
}
```

Apply to specific routes:

```typescript
// features/orders/orders.router.ts
import { idempotency } from '../../shared/middleware/idempotency';

// Idempotency on order creation — prevents duplicate orders
router.post('/', authenticate, idempotency(), validate({ body: createOrderSchema }), controller.create);

// Not needed on GET — already idempotent
router.get('/:id', authenticate, controller.getById);
```

---

## Requiring vs Recommending Idempotency Keys

You have two options: make idempotency keys optional or required for specific endpoints.

**Optional (recommended for most APIs):** If no key is provided, the request is processed normally. Clients that care about safe retries must provide a key; clients that do not care can omit it.

**Required:** The endpoint returns a 400 if no idempotency key is provided. This is the right choice for payment endpoints or any endpoint where duplicate processing is genuinely dangerous.

```typescript
// Middleware variant that requires the key
export function requireIdempotency() {
  return async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    if (req.method === 'POST' && !req.headers['idempotency-key']) {
      next(
        new AppError(
          'INVALID_INPUT',
          'This endpoint requires an Idempotency-Key header. Generate a UUID v4 and include it with the request.',
          400,
        ),
      );
      return;
    }
    // ... rest of logic
  };
}
```

---

## Client-Side Usage

The client is responsible for generating and storing the idempotency key. A UUID v4 is the right format — it is random enough that collisions are astronomically unlikely.

```typescript
// Client-side example (frontend or another service calling your API)
import { v4 as uuidv4 } from 'uuid';

async function createOrder(orderData: CreateOrderDto): Promise<Order> {
  // Generate a key and store it for potential retries
  const idempotencyKey = uuidv4();

  const response = await apiClient.post('/orders', orderData, {
    headers: {
      'Idempotency-Key': idempotencyKey,
    },
  });

  return response.data.data;
}

// With retry logic
async function createOrderWithRetry(orderData: CreateOrderDto): Promise<Order> {
  // Same key across all retry attempts
  const idempotencyKey = uuidv4();
  let lastError: Error;

  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      const response = await apiClient.post('/orders', orderData, {
        headers: { 'Idempotency-Key': idempotencyKey },
      });
      return response.data.data;
    } catch (error) {
      lastError = error as Error;
      // Only retry on network errors or 5xx — never on 4xx
      if (axios.isAxiosError(error) && error.response && error.response.status < 500) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, 1000 * Math.pow(2, attempt)));
    }
  }

  throw lastError!;
}
```

---

## Cleanup

Idempotency records should not be kept forever. Run a cleanup job to delete expired records:

```typescript
// infrastructure/jobs/cleanup-idempotency-records.ts
import { prisma } from '../database/prisma';
import { logger } from '../../shared/utils/logger';

export async function cleanupIdempotencyRecords(): Promise<void> {
  const result = await prisma.idempotencyRecord.deleteMany({
    where: {
      expiresAt: { lt: new Date() },
    },
  });

  logger.info({ count: result.count }, 'Idempotency records cleaned up');
}
```

Run this on a schedule — daily is sufficient for a 24-hour TTL.

---

## Sources

- Stripe API Documentation. "Idempotent Requests." https://stripe.com/docs/api/idempotent_requests — The canonical real-world implementation reference.
- Fielding, Roy, et al. RFC 7231: "HTTP/1.1 Semantics and Content." IETF, 2014. https://datatracker.ietf.org/doc/html/rfc7231 — Idempotency definitions for HTTP methods (Section 4.2.2).
- Kleppmann, Martin. *Designing Data-Intensive Applications*. O'Reilly Media, 2017. — Chapter 12: The trouble with distributed systems and exactly-once semantics.
- AWS Architecture Blog. "Implementing Idempotent APIs." https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/