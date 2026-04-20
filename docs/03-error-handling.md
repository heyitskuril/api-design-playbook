# 3. Error Handling

## The Two Kinds of Errors

Before writing any error handling code, it helps to understand the distinction that Joyent introduced in their Node.js production guide and that has since become the standard framing: **operational errors** vs **programmer errors**.

**Operational errors** are expected. They are conditions that can arise in any correctly-written program: a user tries to access a resource they do not own, a database record is not found, a validation constraint fails, a third-party service is temporarily unavailable. These are not bugs. They are events your application should handle gracefully and return meaningful responses for.

**Programmer errors** are bugs. A `TypeError` because you tried to call `.toLowerCase()` on `undefined`. A database query that throws because you built the WHERE clause wrong. An unhandled promise rejection. These should crash the request loudly (or in Node.js, ideally crash the process and restart it, because continuing with corrupted state is more dangerous than restarting).

Your error handling infrastructure should treat these two categories differently. Operational errors get a specific HTTP status code and a client-readable message. Programmer errors get a 500 with a generic message — the real error goes to your logs and error tracker.

---

## AppError: A Single Error Class

Define one error class that all layers of your application use to communicate operational errors. Every `throw` in your service layer, repository, or middleware should throw an instance of this class.

```typescript
// shared/errors/AppError.ts
export class AppError extends Error {
  public readonly code: string;
  public readonly statusCode: number;
  public readonly isOperational: boolean;
  public readonly details?: unknown;

  constructor(
    code: string,
    message: string,
    statusCode = 500,
    details?: unknown,
  ) {
    super(message);
    this.name = 'AppError';
    this.code = code;
    this.statusCode = statusCode;
    this.isOperational = true;
    this.details = details;

    // Preserves the correct stack trace in V8
    Error.captureStackTrace(this, this.constructor);
  }
}
```

```typescript
// shared/errors/error-codes.ts
// Centralizing error codes prevents typos and makes them discoverable

export const ErrorCodes = {
  // Auth
  UNAUTHORIZED: 'UNAUTHORIZED',
  FORBIDDEN: 'FORBIDDEN',
  TOKEN_EXPIRED: 'TOKEN_EXPIRED',
  INVALID_CREDENTIALS: 'INVALID_CREDENTIALS',

  // Resources
  NOT_FOUND: 'NOT_FOUND',
  ALREADY_EXISTS: 'ALREADY_EXISTS',
  CONFLICT: 'CONFLICT',

  // Validation
  VALIDATION_ERROR: 'VALIDATION_ERROR',
  INVALID_INPUT: 'INVALID_INPUT',

  // Business logic
  INSUFFICIENT_BALANCE: 'INSUFFICIENT_BALANCE',
  ORDER_ALREADY_CANCELLED: 'ORDER_ALREADY_CANCELLED',
  ACCOUNT_NOT_VERIFIED: 'ACCOUNT_NOT_VERIFIED',

  // Rate limiting
  TOO_MANY_REQUESTS: 'TOO_MANY_REQUESTS',

  // Server
  INTERNAL_ERROR: 'INTERNAL_ERROR',
  SERVICE_UNAVAILABLE: 'SERVICE_UNAVAILABLE',
} as const;

export type ErrorCode = typeof ErrorCodes[keyof typeof ErrorCodes];
```

Usage in the service layer:

```typescript
// features/orders/orders.service.ts
import { AppError } from '../../shared/errors/AppError';
import { ErrorCodes } from '../../shared/errors/error-codes';

async getById(orderId: string, requestingUserId: string): Promise<Order> {
  const order = await this.repository.findById(orderId);

  if (!order) {
    throw new AppError(ErrorCodes.NOT_FOUND, 'Order not found.', 404);
  }

  if (order.userId !== requestingUserId) {
    throw new AppError(ErrorCodes.FORBIDDEN, 'You do not have access to this order.', 403);
  }

  return order;
}

async cancel(orderId: string, userId: string): Promise<Order> {
  const order = await this.getById(orderId, userId);

  if (order.status === 'cancelled') {
    throw new AppError(
      ErrorCodes.ORDER_ALREADY_CANCELLED,
      'This order has already been cancelled.',
      409,
    );
  }

  if (['completed', 'shipped'].includes(order.status)) {
    throw new AppError(
      ErrorCodes.CONFLICT,
      `Orders with status "${order.status}" cannot be cancelled.`,
      409,
    );
  }

  return this.repository.update(orderId, { status: 'cancelled' });
}
```

---

## The Global Error Handler

Express has a specific signature for error-handling middleware: four parameters, with the first being `err`. It must be registered after all routes.

```typescript
// shared/middleware/error-handler.ts
import { Request, Response, NextFunction } from 'express';
import { ZodError } from 'zod';
import { AppError } from '../errors/AppError';
import { logger } from '../utils/logger';

export function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction, // Must be declared even if unused — Express needs all 4 parameters
): void {
  const requestId = req.requestId;

  // Zod validation errors — translate to our error format
  if (err instanceof ZodError) {
    const details = err.errors.map((e) => ({
      field: e.path.join('.'),
      message: e.message,
    }));

    res.status(400).json({
      status: 'error',
      code: 'VALIDATION_ERROR',
      message: 'Request validation failed.',
      details,
      requestId,
    });
    return;
  }

  // Operational errors — safe to expose details to the client
  if (err instanceof AppError && err.isOperational) {
    res.status(err.statusCode).json({
      status: 'error',
      code: err.code,
      message: err.message,
      ...(err.details ? { details: err.details } : {}),
      requestId,
    });
    return;
  }

  // Programmer errors / unexpected errors — log fully, expose nothing
  logger.error(
    {
      err: {
        name: err.name,
        message: err.message,
        stack: err.stack,
      },
      request: {
        method: req.method,
        url: req.url,
        requestId,
      },
    },
    'Unexpected error',
  );

  res.status(500).json({
    status: 'error',
    code: 'INTERNAL_ERROR',
    message: 'An unexpected error occurred. Please try again later.',
    requestId,
  });
}
```

Register it in `app.ts`:

```typescript
// src/app.ts
import { errorHandler } from './shared/middleware/error-handler';

// ... all routes ...

// 404 for unknown routes — must come after all valid routes
app.use((req, res) => {
  res.status(404).json({
    status: 'error',
    code: 'NOT_FOUND',
    message: `Route ${req.method} ${req.path} not found.`,
  });
});

// Error handler — must be last, must have 4 parameters
app.use(errorHandler);
```

---

## Error Response Format

Following RFC 9457 (Problem Details for HTTP APIs), your error responses should be consistent and machine-readable. The `code` field is the key: it is a stable, machine-readable identifier that client developers can `switch` on without parsing the human-readable `message`.

```json
// 404 — Resource not found
{
  "status": "error",
  "code": "NOT_FOUND",
  "message": "Order not found.",
  "requestId": "a1b2c3d4-e5f6-..."
}

// 400 — Validation failure
{
  "status": "error",
  "code": "VALIDATION_ERROR",
  "message": "Request validation failed.",
  "details": [
    { "field": "email", "message": "Invalid email address." },
    { "field": "items", "message": "At least one item is required." },
    { "field": "items[0].quantity", "message": "Quantity must be a positive integer." }
  ],
  "requestId": "a1b2c3d4-e5f6-..."
}

// 409 — Business conflict
{
  "status": "error",
  "code": "ORDER_ALREADY_CANCELLED",
  "message": "This order has already been cancelled.",
  "requestId": "a1b2c3d4-e5f6-..."
}

// 429 — Rate limited
{
  "status": "error",
  "code": "TOO_MANY_REQUESTS",
  "message": "Rate limit exceeded. Please retry after 60 seconds.",
  "requestId": "a1b2c3d4-e5f6-..."
}

// 500 — Internal error
{
  "status": "error",
  "code": "INTERNAL_ERROR",
  "message": "An unexpected error occurred. Please try again later.",
  "requestId": "a1b2c3d4-e5f6-..."
}
```

---

## Writing Good Error Messages

Error messages are read by developers integrating your API. Write them accordingly.

**Be specific about what went wrong:**
```
Bad:  "Invalid request."
Good: "The 'quantity' field must be a positive integer."
```

**Be specific about what the client should do:**
```
Bad:  "Authentication failed."
Good: "The provided access token has expired. Request a new token using your refresh token."
```

**Do not expose internal details:**
```
Bad:  "PrismaClientKnownRequestError: Unique constraint failed on the fields: (`email`)"
Good: "An account with this email address already exists."
```

**Use the same language and formality consistently.** If your success messages are formal ("Resource created successfully."), your error messages should be too ("The requested resource was not found."). If they are casual ("Got it — order placed!"), keep errors casual too.

---

## Handling Async Errors in Express

In Express 4, unhandled promise rejections in async route handlers do not automatically reach the error-handling middleware. You must either use a try/catch wrapper or use an async handler utility.

```typescript
// Option A: try/catch in every handler (verbose but explicit)
router.get('/:id', async (req, res, next) => {
  try {
    const order = await service.getById(req.params.id);
    sendSuccess(res, order);
  } catch (error) {
    next(error);
  }
});

// Option B: asyncHandler wrapper (cleaner at scale)
// shared/utils/async-handler.ts
import { Request, Response, NextFunction, RequestHandler } from 'express';

type AsyncRequestHandler = (
  req: Request,
  res: Response,
  next: NextFunction,
) => Promise<void>;

export function asyncHandler(fn: AsyncRequestHandler): RequestHandler {
  return (req, res, next) => {
    fn(req, res, next).catch(next);
  };
}
```

```typescript
// Usage with asyncHandler — much cleaner
router.get(
  '/:id',
  authenticate,
  asyncHandler(async (req, res) => {
    const order = await service.getById(req.params.id, req.user.id);
    sendSuccess(res, order);
  }),
);
```

Note: Express 5 (currently in release candidate) handles async errors automatically. If you are on Express 5, you do not need the `asyncHandler` wrapper.

---

## Sources

- Joyent. "Error Handling in Node.js." https://www.joyent.com/node-js/production/design/errors — The operational vs programmer error distinction.
- Nottingham, Mark, and Erik Wilde. RFC 9457: "Problem Details for HTTP APIs." IETF, 2023. https://datatracker.ietf.org/doc/html/rfc9457
- Nottingham, Mark, and Erik Wilde. RFC 7807: "Problem Details for HTTP APIs." IETF, 2016. https://datatracker.ietf.org/doc/html/rfc7807 — The original, superseded by 9457.
- Nygard, Michael T. *Release It!: Design and Deploy Production-Ready Software*, 2nd ed. Pragmatic Bookshelf, 2018. — Patterns for error handling and stability.
- Express.js Documentation. "Error Handling." https://expressjs.com/en/guide/error-handling.html