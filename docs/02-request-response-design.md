# 2. Request & Response Design

## Why Consistency Matters More Than Any Single Convention

The specific conventions you choose for your response shape matter less than applying them consistently. An API where every endpoint follows the same structure — same field names, same wrapper format, same status code meanings — is dramatically easier to consume than one where each endpoint was designed independently.

The goal of this section is to give you a set of conventions you can apply uniformly. These are not invented here — they are derived from patterns used by Stripe, GitHub, and Google, and from the Problem Details specification (RFC 7807/9457). Where there are alternatives, the trade-offs are noted.

---

## Response Shape

Every response from your API should follow the same envelope structure. There are two schools of thought: bare responses (just the data, no wrapper) and enveloped responses (data wrapped in a consistent object). Both are defensible.

**Bare response:**
```json
{
  "id": "order-123",
  "status": "confirmed",
  "total": 150000
}
```

**Enveloped response:**
```json
{
  "status": "success",
  "data": {
    "id": "order-123",
    "status": "confirmed",
    "total": 150000
  }
}
```

The case for enveloping: it gives you a consistent place to add metadata (pagination cursors, request IDs, warnings) without changing the data structure. It also makes error responses structurally consistent with success responses — both have a `status` field at the top level.

The case against: it adds nesting that clients must unwrap. The HTTP status code already communicates success or failure. Some argue the envelope is redundant.

This guide recommends enveloping. The metadata flexibility is worth the extra nesting, especially as APIs grow. Here is the full shape:

```typescript
// types/api-response.ts

// Success — single resource
interface ResourceResponse<T> {
  status: 'success';
  data: T;
}

// Success — collection with pagination
interface CollectionResponse<T> {
  status: 'success';
  data: T[];
  meta: {
    total?: number;
    nextCursor?: string | null;
    prevCursor?: string | null;
    limit: number;
  };
}

// Error
interface ErrorResponse {
  status: 'error';
  code: string;       // Machine-readable, SCREAMING_SNAKE_CASE
  message: string;    // Human-readable
  details?: ValidationErrorDetail[];  // Field-level errors for validation failures
  requestId?: string; // Trace ID for debugging
}

interface ValidationErrorDetail {
  field: string;
  message: string;
}
```

---

## HTTP Status Codes

Use status codes correctly and consistently. The most common mistake is using 200 for everything and putting the real status in the response body. This breaks HTTP caching, breaks clients that branch on status code, and is incorrect per the HTTP specification.

### Success Codes

| Code | Name | When to use |
|------|------|-------------|
| 200 | OK | GET, PATCH, PUT — successful operation with a body |
| 201 | Created | POST — resource was created. Include Location header pointing to the new resource |
| 204 | No Content | DELETE — successful with no response body |
| 202 | Accepted | POST — request accepted for async processing (not yet complete) |

### Client Error Codes

| Code | Name | When to use |
|------|------|-------------|
| 400 | Bad Request | Malformed request, invalid JSON, failed validation |
| 401 | Unauthorized | No authentication provided, or token is invalid/expired |
| 403 | Forbidden | Authenticated but not authorized for this resource |
| 404 | Not Found | Resource does not exist |
| 409 | Conflict | Business conflict — duplicate email on registration, trying to cancel an already-cancelled order |
| 410 | Gone | Resource existed but has been permanently deleted |
| 422 | Unprocessable Entity | Request is well-formed but semantically invalid — some teams prefer this over 400 for validation errors |
| 429 | Too Many Requests | Rate limit exceeded |

### Server Error Codes

| Code | Name | When to use |
|------|------|-------------|
| 500 | Internal Server Error | Unexpected server-side error |
| 502 | Bad Gateway | Upstream service returned invalid response |
| 503 | Service Unavailable | Server temporarily unavailable (maintenance, overload) |

**On 401 vs 403:** The distinction is important and commonly confused. 401 means "I do not know who you are — authenticate first." 403 means "I know who you are, but you cannot do this." Use them correctly — a client can act on the distinction: retry with credentials on 401, show an access denied message on 403.

---

## Response Helpers in Express

Centralize your response formatting. Do not scatter `res.status(200).json({ status: 'success', data: ... })` across every route handler. A small set of helper functions keeps the format consistent and reduces repetition.

```typescript
// shared/utils/response.ts
import { Response } from 'express';
import { v4 as uuidv4 } from 'uuid';

export function sendSuccess<T>(
  res: Response,
  data: T,
  statusCode = 200,
  headers?: Record<string, string>
): void {
  if (headers) {
    Object.entries(headers).forEach(([key, value]) => res.setHeader(key, value));
  }
  res.status(statusCode).json({ status: 'success', data });
}

export function sendCollection<T>(
  res: Response,
  data: T[],
  meta: { total?: number; nextCursor?: string | null; prevCursor?: string | null; limit: number }
): void {
  res.status(200).json({ status: 'success', data, meta });
}

export function sendCreated<T>(res: Response, data: T, location: string): void {
  res.setHeader('Location', location);
  res.status(201).json({ status: 'success', data });
}

export function sendNoContent(res: Response): void {
  res.status(204).end();
}
```

Usage in a route handler:

```typescript
// features/orders/orders.controller.ts
import { Request, Response, NextFunction } from 'express';
import { sendSuccess, sendCreated, sendNoContent } from '../../shared/utils/response';
import { OrdersService } from './orders.service';

export class OrdersController {
  private service: OrdersService;

  constructor() {
    this.service = new OrdersService();
  }

  getById = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const order = await this.service.getById(req.params.id, req.user.id);
      sendSuccess(res, order);
    } catch (error) {
      next(error);
    }
  };

  create = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      const order = await this.service.create(req.user.id, req.body);
      sendCreated(res, order, `/api/v1/orders/${order.id}`);
    } catch (error) {
      next(error);
    }
  };

  remove = async (req: Request, res: Response, next: NextFunction): Promise<void> => {
    try {
      await this.service.delete(req.params.id, req.user.id);
      sendNoContent(res);
    } catch (error) {
      next(error);
    }
  };
}
```

---

## Field Naming

Use `camelCase` for JSON field names. This is the convention in JavaScript, and since your consumers are overwhelmingly JavaScript/TypeScript applications, it is the path of least friction. The alternative, `snake_case`, is common in Python and Ruby ecosystems.

Whichever you choose, apply it everywhere. Mixing `user_id` in one endpoint and `userId` in another is a sign that the API grew without shared standards.

**Date and time fields** should always use ISO 8601 format in UTC:

```json
{
  "createdAt": "2026-03-15T08:30:00Z",
  "updatedAt": "2026-03-15T09:45:00Z",
  "expiresAt": "2026-04-15T08:30:00Z"
}
```

Never return Unix timestamps as integers. They are not human-readable, they require the client to know the unit (seconds vs milliseconds), and ISO 8601 is the standard.

**Currency and monetary values** should be represented as integers in the smallest unit of the currency:

```json
{
  "total": 150000,
  "currency": "IDR"
}
```

For Indonesian Rupiah, 150000 is Rp 150.000. For USD, 1500 represents $15.00 (cents). This avoids floating-point precision errors in calculations. Stripe uses this convention universally.

**Boolean fields** should use the `is`, `has`, or `can` prefix where it aids readability:

```json
{
  "isVerified": true,
  "hasActiveSubscription": false,
  "canEdit": true
}
```

---

## Request Headers

Your API should accept and validate specific headers:

```typescript
// Standard headers your API should handle
Content-Type: application/json        // Required for POST, PUT, PATCH
Accept: application/json              // Client preference
Authorization: Bearer <token>         // Or handled via cookie
Idempotency-Key: <uuid>               // For POST requests that create resources
X-Request-ID: <uuid>                  // Client-generated request trace ID
```

Set a `X-Request-ID` (or `X-Trace-ID`) on every response, either echoing the client's value or generating one. This allows clients to correlate a specific request with logs on your end when debugging.

```typescript
// shared/middleware/request-id.ts
import { Request, Response, NextFunction } from 'express';
import { v4 as uuidv4 } from 'uuid';

export function requestId(req: Request, res: Response, next: NextFunction): void {
  const id = (req.headers['x-request-id'] as string) || uuidv4();
  req.requestId = id;
  res.setHeader('X-Request-ID', id);
  next();
}

// Extend Express Request type
declare global {
  namespace Express {
    interface Request {
      requestId: string;
    }
  }
}
```

---

## Null vs Omission

When a field has no value, be consistent about whether you send `null` or omit the field entirely.

The recommendation is to send `null` for fields that are part of the resource schema but have no current value. Omitting fields entirely creates uncertainty for the consumer: is this field null, or does this version of the API not return it?

```json
// Consistent — always include fields, use null when empty
{
  "id": "user-123",
  "name": "Kuril",
  "bio": null,
  "avatarUrl": null,
  "verifiedAt": null
}

// Inconsistent — consumer cannot tell if the field exists
{
  "id": "user-123",
  "name": "Kuril"
}
```

The exception is computed or conditional fields that genuinely do not apply — a `refundedAt` field on an order that has never been refunded is null, but a `refundAmount` field on an order that has never been refunded may not exist at all in a context where the order has not gone through the refund flow. Use judgment and document it in your OpenAPI spec.

---

## Sources

- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — Chapters 3 and 4 on request and response design.
- Fielding, Roy, et al. RFC 7231: "Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content." IETF, 2014. https://datatracker.ietf.org/doc/html/rfc7231 — Status code semantics.
- Nottingham, Mark, and Erik Wilde. RFC 9457: "Problem Details for HTTP APIs." IETF, 2023. https://datatracker.ietf.org/doc/html/rfc9457
- ISO 8601: "Date and Time Format." International Organization for Standardization. https://www.iso.org/iso-8601-date-and-time-format.html
- Stripe. "API Reference: Currency and Amounts." https://stripe.com/docs/currencies
- GitHub REST API Documentation. https://docs.github.com/en/rest — Reference for consistent response shapes and status code usage.