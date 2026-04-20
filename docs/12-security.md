# 12. Security

## OWASP API Security Top 10

The Open Worldwide Application Security Project publishes a dedicated API Security Top 10 — separate from the general Web Application Top 10 — because API-specific vulnerabilities have their own distinct character. The 2023 edition is the current reference.

This section covers each of the ten risks as they apply to an Express/Node.js API, with concrete mitigations.

---

### API1: Broken Object Level Authorization (BOLA)

The most common API vulnerability. BOLA occurs when an API endpoint accepts a resource identifier (an order ID, user ID, document ID) from the client but does not verify that the authenticated user is authorized to access that specific object.

```typescript
// Vulnerable
router.get('/:id', authenticate, async (req, res) => {
  const order = await ordersRepository.findById(req.params.id);
  // No check: does this order belong to req.user.id?
  res.json(order);
});

// Fixed
router.get('/:id', authenticate, asyncHandler(async (req, res) => {
  const order = await ordersService.getById(req.params.id, req.user.id);
  // Service throws 404 if order belongs to another user
  sendSuccess(res, order);
}));
```

Every endpoint that accepts a resource ID must verify ownership or authorization in the service layer. This check cannot be skipped even for "internal" endpoints.

---

### API2: Broken Authentication

Weak authentication mechanisms, exposed credentials, and improperly implemented token handling.

**Mitigations covered in this guide:**
- JWT stored in `httpOnly` cookies, not `localStorage` (Section 7)
- Short access token lifetime (15 minutes) with refresh token rotation (Section 7)
- Rate limiting on authentication endpoints (Section 9)
- Consistent error messages for login failures — "Invalid email or password" for both wrong email and wrong password to prevent user enumeration (Section 7)
- bcrypt with adequate cost factor for password hashing (Section 7)

```typescript
// Never expose which part of the login failed
// Bad — enables user enumeration
if (!user) throw new AppError('USER_NOT_FOUND', 'No account with this email.', 404);
if (!passwordMatch) throw new AppError('WRONG_PASSWORD', 'Incorrect password.', 401);

// Good — same message for both cases
if (!user || !(await bcrypt.compare(dto.password, user.passwordHash))) {
  throw new AppError('INVALID_CREDENTIALS', 'Invalid email or password.', 401);
}
```

---

### API3: Broken Object Property Level Authorization

Exposing more properties than the client should see, or allowing clients to modify properties they should not be able to. This is the "mass assignment" vulnerability.

```typescript
// Vulnerable — passes entire req.body to the repository
router.patch('/me', authenticate, async (req, res) => {
  const user = await usersRepository.update(req.user.id, req.body);
  // Client could send { role: 'admin', isVerified: true } in the body
  res.json(user);
});

// Fixed — use a strict Zod schema that only allows specific fields
export const updateUserSchema = z.object({
  name: z.string().min(1).max(100).optional(),
  bio: z.string().max(500).optional(),
  // 'role' and 'isVerified' are not in this schema — they cannot be set here
});

router.patch('/me', authenticate, validate({ body: updateUserSchema }), asyncHandler(async (req, res) => {
  const user = await usersService.update(req.user.id, req.body); // Only validated fields
  sendSuccess(res, user);
}));
```

Also never return sensitive internal fields (password hash, internal flags, tokens) in API responses. Define explicit response types that include only what clients need:

```typescript
// Always pick specific fields — never return the full Prisma model
async function toPublicUser(user: User) {
  return {
    id: user.id,
    name: user.name,
    email: user.email,
    role: user.role,
    createdAt: user.createdAt,
    // passwordHash, internalNotes, etc. are not here
  };
}
```

---

### API4: Unrestricted Resource Consumption

No limits on how much data, bandwidth, or computation a client can consume.

**Mitigations:**
- Pagination with a maximum page size (Section 5): `max: 100` in the limit parameter
- Rate limiting (Section 9)
- Request body size limits in Express:

```typescript
// Limit request body size globally
app.use(express.json({ limit: '100kb' }));
app.use(express.urlencoded({ extended: true, limit: '100kb' }));

// Stricter limit for specific endpoints
app.use('/api/v1/uploads', express.json({ limit: '10mb' }));
```

- File upload validation — check file type and size before processing
- Set timeouts on external service calls and database queries

---

### API5: Broken Function Level Authorization

Endpoints that perform privileged actions are accessible to users without the required role.

```typescript
// All admin endpoints must have role check — not just authentication
router.get('/admin/users', authenticate, authorize('admin'), controller.listAllUsers);
router.delete('/admin/users/:id', authenticate, authorize('admin'), controller.deleteUser);
router.get('/reports/financial', authenticate, authorize('admin', 'finance'), controller.financialReport);

// Deny by default — unlisted roles cannot access the endpoint
export function authorize(...roles: string[]) {
  return (req: Request, res: Response, next: NextFunction) => {
    if (!roles.includes(req.user.role)) {
      next(new AppError('FORBIDDEN', 'Insufficient permissions.', 403));
      return;
    }
    next();
  };
}
```

---

### API6: Unrestricted Access to Sensitive Business Flows

Business flows that can be abused if not rate-limited or protected — account registration bots, coupon code exhaustion, automated purchasing.

```typescript
// Specific limits for sensitive business flows
export const registrationLimiter = rateLimit({
  windowMs: 60 * 60 * 1000, // 1 hour
  max: 5,                    // 5 registrations per hour per IP
  standardHeaders: 'draft-7',
  message: { status: 'error', code: 'TOO_MANY_REQUESTS', message: 'Registration limit reached.' },
});

// CAPTCHA for bot-prone flows
// Consider server-side CAPTCHA verification for registration, contact forms
```

---

### API7: Server-Side Request Forgery (SSRF)

The server makes an HTTP request to a URL supplied by the client, potentially allowing access to internal services.

If your API accepts URLs from clients (webhooks, avatar URLs, link previews), validate them:

```typescript
import { URL } from 'url';

function validateExternalUrl(url: string): void {
  let parsed: URL;
  try {
    parsed = new URL(url);
  } catch {
    throw new AppError('INVALID_INPUT', 'Invalid URL format.', 400);
  }

  // Only allow HTTPS
  if (parsed.protocol !== 'https:') {
    throw new AppError('INVALID_INPUT', 'Only HTTPS URLs are allowed.', 400);
  }

  // Block internal IP ranges
  const blockedPatterns = [
    /^localhost$/i,
    /^127\./,
    /^10\./,
    /^172\.(1[6-9]|2[0-9]|3[0-1])\./,
    /^192\.168\./,
    /^169\.254\./, // Link-local
    /^::1$/,       // IPv6 localhost
  ];

  if (blockedPatterns.some((pattern) => pattern.test(parsed.hostname))) {
    throw new AppError('INVALID_INPUT', 'URL refers to an internal address.', 400);
  }
}
```

---

### API8: Security Misconfiguration

Default configurations, verbose error messages, missing headers, open CORS.

```typescript
// Helmet — sets security headers
import helmet from 'helmet';
app.use(helmet({
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      // ... configure for your specific needs
    },
  },
}));

// CORS — whitelist only known origins
import cors from 'cors';
app.use(cors({
  origin: (origin, callback) => {
    const allowed = env.ALLOWED_ORIGINS?.split(',').map((o) => o.trim()) ?? [];
    if (!origin || allowed.includes(origin)) {
      callback(null, true);
    } else {
      callback(new Error('Not allowed by CORS'));
    }
  },
  credentials: true,
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
}));

// Never expose stack traces or internal error details in production
// The error handler in Section 3 handles this correctly
```

---

### API9: Improper Inventory Management

Outdated or undocumented API versions still accessible in production. APIs exposed on unexpected paths.

- Maintain an accurate OpenAPI spec (Section 10)
- Formally deprecate old API versions with `Deprecation` and `Sunset` headers (Section 4)
- Remove test and debug endpoints before going to production
- Audit your route list periodically against what is actually needed

---

### API10: Unsafe Consumption of APIs

Trusting data from third-party APIs without validation.

```typescript
// Validate data from external APIs before using it
import { z } from 'zod';

const paymentWebhookSchema = z.object({
  event: z.enum(['payment.succeeded', 'payment.failed', 'payment.refunded']),
  data: z.object({
    orderId: z.string().uuid(),
    amount: z.number().positive(),
    currency: z.string().length(3),
  }),
});

router.post('/webhooks/payment', async (req, res, next) => {
  // Verify webhook signature before processing
  const signature = req.headers['x-payment-signature'] as string;
  if (!verifySignature(req.rawBody, signature, env.PAYMENT_WEBHOOK_SECRET)) {
    return res.status(401).end();
  }

  const result = paymentWebhookSchema.safeParse(req.body);
  if (!result.success) {
    // Log invalid webhook but return 200 to avoid retries
    logger.warn({ body: req.body }, 'Invalid webhook payload received');
    return res.status(200).end();
  }

  await webhookService.process(result.data);
  res.status(200).end();
});
```

---

## Security Headers Checklist

After applying Helmet, verify your response headers with [securityheaders.com](https://securityheaders.com):

| Header | Purpose | Expected value |
|--------|---------|----------------|
| `Strict-Transport-Security` | Force HTTPS | `max-age=31536000; includeSubDomains` |
| `Content-Security-Policy` | Prevent XSS | Tailored to your content |
| `X-Content-Type-Options` | Prevent MIME sniffing | `nosniff` |
| `X-Frame-Options` | Prevent clickjacking | `DENY` |
| `Referrer-Policy` | Limit referrer info | `strict-origin-when-cross-origin` |
| `Permissions-Policy` | Disable browser features | `geolocation=(), microphone=()` |

---

## Sources

- OWASP. "OWASP API Security Top 10 — 2023." https://owasp.org/API-Security/editions/2023/en/0x00-header/
- OWASP. "REST Security Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html
- OWASP. "Mass Assignment Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html
- OWASP. "Server Side Request Forgery Prevention Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html
- Helmet.js Documentation. https://helmetjs.github.io/
- Mozilla Observatory. "HTTP Observatory." https://observatory.mozilla.org/ — Free security scanner.
- Security Headers Scanner. https://securityheaders.com/