# 7. Authentication & Authorization in APIs

## Auth in APIs vs Auth in Web Applications

The auth section in the PERN Stack Architecture Guide covers the full implementation in detail, including JWT rotation, httpOnly cookies, and RBAC. This section focuses specifically on the API-layer concerns: how auth is expressed in your API design, how to handle auth in different client contexts, and how authorization decisions are made at the endpoint level.

If you are building an API consumed only by your own frontend, the PERN architecture guide's approach (httpOnly cookies, JWT with refresh rotation) is the right choice. This section also covers API key authentication, which is relevant for server-to-server integrations and developer-facing APIs.

---

## Bearer Token Authentication

For APIs consumed by clients that cannot use cookies — mobile apps, server-side applications, CLI tools — Bearer tokens in the `Authorization` header are the standard:

```
GET /api/v1/orders
Authorization: Bearer eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

The middleware reads the header, verifies the JWT, and attaches the decoded payload to `req.user`:

```typescript
// shared/middleware/authenticate.ts
import { Request, Response, NextFunction } from 'express';
import jwt from 'jsonwebtoken';
import { env } from '../../config/env';
import { AppError } from '../errors/AppError';

interface JwtPayload {
  sub: string;
  role: string;
  iat: number;
  exp: number;
}

declare global {
  namespace Express {
    interface Request {
      user: {
        id: string;
        role: string;
      };
    }
  }
}

export function authenticate(req: Request, res: Response, next: NextFunction): void {
  // Support both cookie-based (web frontend) and Bearer token (API clients)
  let token: string | undefined;

  const authHeader = req.headers.authorization;
  if (authHeader?.startsWith('Bearer ')) {
    token = authHeader.slice(7);
  } else if (req.cookies?.accessToken) {
    token = req.cookies.accessToken;
  }

  if (!token) {
    next(new AppError('UNAUTHORIZED', 'Authentication required.', 401));
    return;
  }

  try {
    const payload = jwt.verify(token, env.JWT_ACCESS_SECRET) as JwtPayload;
    req.user = { id: payload.sub, role: payload.role };
    next();
  } catch (error) {
    if (error instanceof jwt.TokenExpiredError) {
      next(new AppError('TOKEN_EXPIRED', 'Access token has expired. Please refresh your token.', 401));
    } else {
      next(new AppError('UNAUTHORIZED', 'Invalid access token.', 401));
    }
  }
}

export function optionalAuthenticate(req: Request, res: Response, next: NextFunction): void {
  // For endpoints that work with or without authentication
  // (e.g., public product listing that shows extra data when logged in)
  try {
    const authHeader = req.headers.authorization;
    const token = authHeader?.startsWith('Bearer ') ? authHeader.slice(7) : req.cookies?.accessToken;

    if (token) {
      const payload = jwt.verify(token, env.JWT_ACCESS_SECRET) as JwtPayload;
      req.user = { id: payload.sub, role: payload.role };
    }
  } catch {
    // Silent — unauthenticated requests are fine for this middleware
  }
  next();
}
```

---

## API Key Authentication

For server-to-server integrations — a third party's backend calling your API, a webhook consumer, or an internal service — API keys are often more appropriate than JWTs. A key is a long random string, issued once, stored securely by the holder, and sent with every request.

```
GET /api/v1/products
X-API-Key: sk_live_abc123def456...
```

### Generating API Keys

```typescript
// features/api-keys/api-keys.service.ts
import crypto from 'crypto';
import bcrypt from 'bcrypt';

interface ApiKey {
  key: string;      // The full key — shown to the user once, never stored again
  prefix: string;   // First 8 chars — stored, used to look up the key
  hash: string;     // bcrypt hash of the full key — stored in database
}

export async function generateApiKey(): Promise<ApiKey> {
  // Generate a 32-byte random key, encode as hex
  const rawKey = crypto.randomBytes(32).toString('hex');
  const prefix = rawKey.slice(0, 8);
  const key = `sk_${prefix}_${rawKey}`;

  // Hash the full key before storing
  const hash = await bcrypt.hash(key, 12);

  return { key, prefix, hash };
}
```

### Validating API Keys

```typescript
// shared/middleware/authenticate-api-key.ts
import { Request, Response, NextFunction } from 'express';
import bcrypt from 'bcrypt';
import { prisma } from '../../infrastructure/database/prisma';
import { AppError } from '../errors/AppError';

export async function authenticateApiKey(
  req: Request,
  res: Response,
  next: NextFunction,
): Promise<void> {
  const apiKey = req.headers['x-api-key'] as string | undefined;

  if (!apiKey) {
    next(new AppError('UNAUTHORIZED', 'API key required.', 401));
    return;
  }

  // Extract prefix to find the right record (avoids full table scan)
  const prefix = apiKey.slice(3, 11); // After "sk_"

  const record = await prisma.apiKey.findFirst({
    where: { prefix, revokedAt: null },
    include: { owner: true },
  });

  if (!record) {
    next(new AppError('UNAUTHORIZED', 'Invalid API key.', 401));
    return;
  }

  const valid = await bcrypt.compare(apiKey, record.hash);

  if (!valid) {
    next(new AppError('UNAUTHORIZED', 'Invalid API key.', 401));
    return;
  }

  // Update last used timestamp asynchronously
  prisma.apiKey
    .update({ where: { id: record.id }, data: { lastUsedAt: new Date() } })
    .catch(() => {}); // Non-blocking, non-critical

  req.user = { id: record.owner.id, role: record.owner.role };
  next();
}
```

Note: bcrypt comparison on every request is expensive. For high-traffic APIs, consider caching validated API keys in Redis with a short TTL (e.g., 5 minutes):

```typescript
// With Redis cache
const cacheKey = `api-key:${prefix}`;
const cached = await redis.get(cacheKey);

if (cached) {
  req.user = JSON.parse(cached);
  next();
  return;
}

// ... validate against database ...
// On success:
await redis.setex(cacheKey, 300, JSON.stringify(req.user)); // Cache for 5 minutes
```

---

## Authorization at the Endpoint Level

Authentication answers "who are you." Authorization answers "what are you allowed to do." Both happen at different places in the request lifecycle.

Role-based middleware handles coarse-grained access — only admins can access this endpoint:

```typescript
// shared/middleware/authorize.ts
export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction): void => {
    if (!roles.includes(req.user.role)) {
      next(new AppError('FORBIDDEN', 'Insufficient permissions for this action.', 403));
      return;
    }
    next();
  };
}

// Usage
router.get('/admin/users', authenticate, authorize('admin'), controller.listAllUsers);
router.get('/reports', authenticate, authorize('admin', 'editor'), controller.getReports);
```

Fine-grained authorization — "this user can only access their own orders" — belongs in the service layer, not in middleware. The service has access to business data (who owns this order?) that middleware does not:

```typescript
// In the service layer
async getOrder(orderId: string, requestingUserId: string, requestingRole: string): Promise<Order> {
  const order = await this.repository.findById(orderId);

  if (!order) {
    throw new AppError('NOT_FOUND', 'Order not found.', 404);
  }

  // Admins can access any order; users can only access their own
  if (requestingRole !== 'admin' && order.userId !== requestingUserId) {
    // Return 404, not 403 — do not confirm that the resource exists
    // This prevents resource enumeration attacks
    throw new AppError('NOT_FOUND', 'Order not found.', 404);
  }

  return order;
}
```

The decision to return 404 instead of 403 for resources the user does not own is a security practice: returning 403 confirms that the resource exists, which leaks information. For most cases, 404 is the safer choice.

---

## Documenting Auth in Your API

Every endpoint's authentication requirement should be documented explicitly. In your OpenAPI spec (Section 10), use security schemes:

```yaml
# openapi.yaml
components:
  securitySchemes:
    BearerAuth:
      type: http
      scheme: bearer
      bearerFormat: JWT
    ApiKeyAuth:
      type: apiKey
      in: header
      name: X-API-Key

paths:
  /orders:
    get:
      security:
        - BearerAuth: []
        - ApiKeyAuth: []    # Either works
      ...
  /public/products:
    get:
      security: []          # No auth required — explicitly stated
      ...
```

---

## Sources

- OWASP. "Authentication Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html
- OWASP. "REST Security Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- IETF. RFC 8725: "JSON Web Token Best Current Practices." 2020. https://datatracker.ietf.org/doc/html/rfc8725
- Stripe API Documentation. "Authentication." https://stripe.com/docs/api/authentication — API key design reference.
- Richer, Justin, and Antonio Sanso. *OAuth 2 in Action*. Manning Publications, 2017.