# 4. Versioning

## Why You Need a Versioning Strategy Before You Launch

Once an API is in production with real consumers, breaking changes become expensive. Removing a field, renaming a property, changing a type, altering the meaning of a status code — any of these can break a client application you do not control. A versioning strategy defines how you will introduce breaking changes without breaking existing consumers.

The time to decide on a versioning strategy is before your first public release, not after. Retrofitting versioning onto an existing API is painful.

---

## What Counts as a Breaking Change

Understanding what is and is not breaking helps you decide when versioning is actually needed. Not every API change requires a new version.

**Breaking changes — require a version bump:**
- Removing a field from a response
- Renaming a field
- Changing a field's type (`string` → `integer`)
- Changing a field's format (`ISO 8601` → `Unix timestamp`)
- Changing the meaning of an HTTP status code
- Removing an endpoint
- Changing required request fields
- Changing the behavior of an existing endpoint in a way that alters its observable output

**Non-breaking changes — safe to deploy without a version bump:**
- Adding a new optional field to a response
- Adding a new optional request parameter
- Adding a new endpoint
- Adding a new error code (as long as clients handle unknown codes gracefully)
- Performance improvements with no observable behavioral change

This means you can and should design your clients to be **tolerant of unknown fields**. A client that throws an error when it encounters a JSON key it does not recognize is fragile. Postel's Law: be liberal in what you accept.

---

## Versioning Strategies

There are three main approaches. Each has genuine trade-offs.

### URL Path Versioning

```
GET /api/v1/orders
GET /api/v2/orders
```

This is the most widely used approach. It is explicit, visible, cacheable, and easy to reason about. A request to `/api/v1/orders` will always behave like a v1 request, regardless of headers or query parameters.

**Pros:** Easy to route, easy to debug, works everywhere (CDNs, proxies, logs), immediately obvious to anyone reading a URL.

**Cons:** Violates the REST principle that a URL should identify a resource (not a version of a resource). Requires clients to update the URL when migrating versions.

This is the approach used by Stripe, GitHub, Twitter/X, and most major APIs. The theoretical purity objection is real, but the practical advantages outweigh it.

### Header Versioning

```
GET /api/orders
Accept: application/vnd.example.v2+json
```

Or using a custom header:
```
GET /api/orders
API-Version: 2
```

**Pros:** Cleaner URLs. Aligns with REST's uniform interface — the URL identifies the resource, the header specifies the representation.

**Cons:** Harder to test (you cannot share a URL), does not work in a browser address bar, harder to route at the infrastructure level, easier to forget. Many caching intermediaries ignore custom headers, which means versioned responses may be incorrectly cached.

### Query Parameter Versioning

```
GET /api/orders?version=2
```

**Pros:** Easy to implement, easy to see in logs.

**Cons:** Versions are often accidentally omitted, query parameters are typically reserved for filtering and pagination, creating confusion.

**Recommendation:** Use URL path versioning. The practical benefits — debuggability, routability, shareability — outweigh the theoretical concerns about URL purity. If you are building a public API that other developers will use, this is the right choice.

---

## Implementation in Express

With URL versioning, organize your Express routers under versioned path prefixes:

```typescript
// src/app.ts
import v1Router from './api/v1/router';
import v2Router from './api/v2/router';

app.use('/api/v1', v1Router);
app.use('/api/v2', v2Router);
```

```typescript
// src/api/v1/router.ts
import { Router } from 'express';
import ordersRouter from '../../features/orders/orders.router';
import usersRouter from '../../features/users/users.router';

const router = Router();

router.use('/orders', ordersRouter);
router.use('/users', usersRouter);

export default router;
```

When introducing v2, you do not rewrite everything. Most endpoints stay identical. Only the changed endpoints get v2-specific implementations:

```typescript
// src/api/v2/router.ts
import { Router } from 'express';
import ordersRouterV2 from '../../features/orders/orders.router.v2'; // Changed
import usersRouter from '../../features/users/users.router'; // Unchanged — reuse v1

const router = Router();

router.use('/orders', ordersRouterV2);
router.use('/users', usersRouter); // Same handler, different base path

export default router;
```

This avoids duplicating code for every endpoint just because one endpoint changed.

---

## When to Create a New Version

The bar for creating a new API version should be high. A new version means:

- Existing consumers must be notified
- You must maintain both versions in parallel for a defined deprecation period
- Documentation must cover both versions
- Test coverage must cover both versions

Given that cost, push hard for backwards-compatible changes before creating a new version. Can you add the new field alongside the old one? Can you accept both formats and normalize internally? Can you add a new endpoint rather than changing an existing one?

Stripe has maintained `v1` of their API since 2011. They introduce breaking changes through a versioning date system (e.g., `2024-04-10`) rather than major version numbers, allowing granular migration. This is sophisticated and worth understanding for large APIs, though for most applications a simple `v1`/`v2` scheme is sufficient.

---

## Deprecation

When you decide to deprecate an old version:

**Give notice in advance.** Six months is a reasonable minimum for production APIs with external consumers. Announce the deprecation date clearly in your documentation and changelog.

**Add deprecation headers to responses from deprecated versions:**

```typescript
// middleware for deprecated routes
export function deprecated(sunsetDate: string) {
  return (req: Request, res: Response, next: NextFunction) => {
    res.setHeader('Deprecation', 'true');
    res.setHeader('Sunset', sunsetDate); // RFC 8594 — the date the version will stop working
    res.setHeader(
      'Link',
      '<https://api.example.com/api/v2/orders>; rel="successor-version"',
    );
    next();
  };
}
```

```typescript
// Apply to the entire v1 router when deprecating it
app.use('/api/v1', deprecated('2026-01-01'), v1Router);
```

The `Sunset` header (RFC 8594) is the standard way to communicate the exact date a deprecated API version will cease to function.

**Log access to deprecated endpoints.** You need to know which clients are still using the old version so you can reach out before the sunset date.

---

## Versioning Internal vs External APIs

If your API is only consumed internally — by your own frontend, by your own services — the stakes of breaking changes are lower because you control all consumers. You can update both the API and all clients in a single deployment.

In this case, a strict versioning strategy may be overkill. Good practices like maintaining backwards compatibility, communicating changes in your team, and using contract tests (see Section 11) can substitute for formal versioning.

Formal API versioning is most important for public APIs or APIs consumed by third parties you do not control.

---

## Sources

- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — Chapter 9: Evolving an API.
- Fielding, Roy, et al. RFC 7231: "HTTP/1.1 Semantics and Content." IETF, 2014. https://datatracker.ietf.org/doc/html/rfc7231
- Wilde, Erik. RFC 8594: "The Sunset HTTP Header Field." IETF, 2019. https://datatracker.ietf.org/doc/html/rfc8594
- Stripe Engineering Blog. "API Versioning." https://stripe.com/blog/api-versioning — How Stripe manages API evolution at scale.
- Troy Hunt. "Your API versioning is wrong, which is why I decided to do it 3 different wrong ways." https://www.troyhunt.com/your-api-versioning-is-wrong-which-is — A clear comparison of trade-offs.
- Microsoft. "REST API Design — Versioning." https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design#versioning-a-restful-web-api