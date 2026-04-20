# 9. Rate Limiting & Throttling

## Why Rate Limiting Is Not Optional

Rate limiting serves three purposes: protecting your server from being overwhelmed by a single client (intentional or not), preventing brute-force attacks on sensitive endpoints, and enforcing fair use across multiple consumers of a shared API.

Without rate limiting, a single client with a bug in their retry logic — or a single attacker running a script — can consume all your server resources. This is not a hypothetical; it is common enough that most production APIs treat rate limiting as a baseline requirement, not a premium feature.

---

## Rate Limiting Strategies

### Fixed Window

Counts requests within a fixed time window (e.g., 100 requests per minute). When the window expires, the count resets. Simple to implement and understand.

**Weakness:** Allows burst traffic at window boundaries. A client can make 100 requests in the last second of one window and 100 requests in the first second of the next window — 200 requests in 2 seconds while technically staying within limits.

### Sliding Window

Counts requests within a rolling window relative to the current time. More accurate than fixed window and prevents boundary bursting.

**Implementation:** Track timestamps of recent requests and count how many fall within the window. More complex but significantly fairer.

### Token Bucket

Tokens are added to a bucket at a fixed rate (the refill rate). Each request consumes one token. If the bucket is empty, the request is rejected. The bucket has a maximum capacity (the burst limit).

This allows short bursts above the average rate (up to the bucket size) while enforcing a long-term average rate. It is how most production rate limiters, including AWS API Gateway and Stripe, work.

### Leaky Bucket

Requests enter a queue and are processed at a fixed rate regardless of input rate. Excess requests either wait or are dropped. Smooths out traffic spikes. Used more often in traffic shaping than in API limiting.

For most API rate limiting, **sliding window or token bucket** are the right choices.

---

## Implementation with Express

### Basic Rate Limiting with `express-rate-limit`

```typescript
import rateLimit from 'express-rate-limit';
import { Request, Response } from 'express';

// General API rate limiter — apply globally
export const apiLimiter = rateLimit({
  windowMs: 60 * 1000,     // 1 minute
  max: 100,                // 100 requests per window per IP
  standardHeaders: 'draft-7', // Return `RateLimit-*` headers per RFC draft 7
  legacyHeaders: false,
  keyGenerator: (req: Request) => {
    // Rate limit by authenticated user ID when available, otherwise by IP
    return req.user?.id ?? req.ip ?? 'unknown';
  },
  handler: (req: Request, res: Response) => {
    res.status(429).json({
      status: 'error',
      code: 'TOO_MANY_REQUESTS',
      message: 'Rate limit exceeded. Please wait before making additional requests.',
      requestId: req.requestId,
    });
  },
  skip: (req: Request) => {
    // Skip rate limiting for internal health checks
    return req.path === '/health';
  },
});

// Strict limiter for auth endpoints — brute force protection
export const authLimiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 10,                   // 10 attempts per 15 minutes per IP
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  skipSuccessfulRequests: true, // Only count failed attempts
  keyGenerator: (req: Request) => req.ip ?? 'unknown',
  handler: (req: Request, res: Response) => {
    res.status(429).json({
      status: 'error',
      code: 'TOO_MANY_REQUESTS',
      message: 'Too many login attempts. Please try again in 15 minutes.',
      requestId: req.requestId,
    });
  },
});

// Lenient limiter for public read endpoints
export const publicLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 300,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
});
```

Apply in `app.ts` and on specific routes:

```typescript
// Global — all API routes
app.use('/api/', apiLimiter);

// Strict — auth endpoints
app.use('/api/v1/auth/login', authLimiter);
app.use('/api/v1/auth/register', authLimiter);
app.use('/api/v1/auth/forgot-password', authLimiter);

// Lenient — public product catalog
app.use('/api/v1/products', publicLimiter);
```

### Redis-Backed Rate Limiting (Production)

The default in-memory store for `express-rate-limit` does not work correctly when you run multiple server instances — each instance has its own counter, so limits are effectively multiplied. In production with load balancing, use a Redis store:

```typescript
import rateLimit from 'express-rate-limit';
import { RedisStore } from 'rate-limit-redis';
import { redis } from '../../infrastructure/cache/redis';

export const apiLimiter = rateLimit({
  windowMs: 60 * 1000,
  max: 100,
  standardHeaders: 'draft-7',
  legacyHeaders: false,
  store: new RedisStore({
    sendCommand: (...args: string[]) => redis.call(...args),
    prefix: 'rl:api:',
  }),
  keyGenerator: (req) => req.user?.id ?? req.ip ?? 'unknown',
  handler: (req, res) => {
    res.status(429).json({
      status: 'error',
      code: 'TOO_MANY_REQUESTS',
      message: 'Rate limit exceeded.',
      requestId: req.requestId,
    });
  },
});
```

```typescript
// infrastructure/cache/redis.ts
import Redis from 'ioredis';
import { env } from '../../config/env';

export const redis = new Redis(env.REDIS_URL, {
  maxRetriesPerRequest: 3,
  retryStrategy: (times: number) => Math.min(times * 50, 2000),
});

redis.on('error', (err) => {
  console.error('Redis connection error:', err);
});
```

---

## Rate Limit Response Headers

Tell clients what their limit is and when they can retry. The emerging standard is the `RateLimit-*` header set from RFC draft 7 (which `express-rate-limit` with `standardHeaders: 'draft-7'` provides):

```
RateLimit-Limit: 100
RateLimit-Remaining: 73
RateLimit-Reset: 1742400000     (Unix timestamp when the window resets)
Retry-After: 60                 (Seconds to wait — returned only on 429)
```

Older APIs use `X-RateLimit-*` (the "legacy headers"). The standard is moving toward `RateLimit-*` without the `X-` prefix. Use the standard form in new APIs.

---

## Tiered Rate Limits

Different client types get different limits. API key clients are usually more trusted than anonymous clients. Enterprise customers on a paid plan may have higher limits.

```typescript
export function tieredLimiter() {
  const limits = {
    anonymous: { windowMs: 60_000, max: 30 },
    authenticated: { windowMs: 60_000, max: 100 },
    premium: { windowMs: 60_000, max: 1000 },
    internal: { windowMs: 60_000, max: 10_000 },
  };

  return rateLimit({
    windowMs: 60_000,
    max: (req) => {
      if (!req.user) return limits.anonymous.max;
      if (req.user.plan === 'premium') return limits.premium.max;
      if (req.user.role === 'internal') return limits.internal.max;
      return limits.authenticated.max;
    },
    standardHeaders: 'draft-7',
    legacyHeaders: false,
    store: new RedisStore({
      sendCommand: (...args: string[]) => redis.call(...args),
      prefix: 'rl:tiered:',
    }),
    keyGenerator: (req) => req.user?.id ?? req.ip ?? 'unknown',
  });
}
```

---

## Handling 429 on the Client Side

Your API documentation should explain what clients should do when they receive a 429:

```typescript
// Client-side: respect Retry-After header
async function callApiWithRetry<T>(request: () => Promise<T>): Promise<T> {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await request();
    } catch (error) {
      if (axios.isAxiosError(error) && error.response?.status === 429) {
        const retryAfter = Number(error.response.headers['retry-after']) || 60;
        await new Promise((resolve) => setTimeout(resolve, retryAfter * 1000));
        continue;
      }
      throw error;
    }
  }
  throw new Error('Max retries reached.');
}
```

---

## Sources

- OWASP. "Blocking Brute Force Attacks." https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks
- Ietf. "RateLimit Header Fields for HTTP." Internet-Draft. https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/
- express-rate-limit Documentation. https://github.com/express-rate-limit/express-rate-limit
- Redis Documentation. "Rate Limiting with Redis." https://redis.io/learn/develop/dotnet/aspnetcore/rate-limiting/sliding-window
- Kleppmann, Martin. *Designing Data-Intensive Applications*. O'Reilly Media, 2017. — Chapter 1: Scalability — load parameters and throughput.