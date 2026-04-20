# 1. REST Fundamentals

## What REST Actually Is

REST is short for Representational State Transfer. It was defined in Roy Fielding's 2000 doctoral dissertation at UC Irvine — not invented by a committee, not a specification document you can look up and follow line by line, but an architectural style described as a set of constraints. If an API satisfies those constraints, it is RESTful. If it does not, it is not, regardless of whether it uses HTTP.

This matters because a lot of APIs call themselves REST when they are actually RPC-over-HTTP — which is not inherently wrong, but it is worth being honest about what you are building.

The six constraints Fielding defined:

**Client-server separation.** The client and server evolve independently. The client does not care how the server stores data. The server does not care how the client renders it.

**Statelessness.** Each request from the client contains all the information the server needs to process it. The server holds no session state between requests. This is what makes horizontal scaling straightforward — any server instance can handle any request.

**Cacheability.** Responses must define whether they are cacheable. This allows clients and intermediaries (CDNs, proxies) to cache responses, reducing load and latency.

**Uniform interface.** The interface between client and server follows consistent conventions — resource identification through URIs, resource manipulation through representations, self-descriptive messages, and hypermedia as the engine of application state (HATEOAS). In practice, most APIs implement the first three and skip HATEOAS. That is a reasonable trade-off.

**Layered system.** The client does not know whether it is talking directly to the server or to a proxy, load balancer, or cache layer. Each layer only sees the layer immediately adjacent to it.

**Code on demand (optional).** Servers can send executable code to clients. Rarely used and not required.

---

## HTTP Semantics: Using the Right Method

The HTTP specification (RFC 7231) defines the semantics of HTTP methods. These are not suggestions. Violating them breaks caching, breaks idempotency guarantees, and breaks the expectations of anyone consuming your API.

| Method | Semantics | Safe | Idempotent | Request body |
|--------|-----------|------|------------|--------------|
| GET | Retrieve a resource | Yes | Yes | No |
| POST | Create a resource or trigger an action | No | No | Yes |
| PUT | Replace a resource entirely | No | Yes | Yes |
| PATCH | Partially update a resource | No | No | Yes |
| DELETE | Remove a resource | No | Yes | No |
| HEAD | Same as GET but no body | Yes | Yes | No |
| OPTIONS | Describe communication options | Yes | Yes | No |

**Safe** means the request has no observable side effects. A GET request should never modify state.

**Idempotent** means sending the same request multiple times produces the same result as sending it once. PUT and DELETE are idempotent: deleting a resource twice leaves you in the same state as deleting it once (the resource is gone). POST is not: creating the same resource twice creates two resources.

These properties have real consequences. Browsers retry safe requests automatically. CDNs cache safe requests. HTTP clients may retry idempotent requests on network failure. If you use POST for operations that should be idempotent, you lose these behaviors.

---

## Resource Modeling

Resources are the nouns of your API. A resource represents a thing — a user, an order, a product, a payment — not an action.

The most common mistake in API design is modeling resources around actions instead of around entities. This leads to URLs like:

```
POST /api/getUserById
POST /api/createOrder
POST /api/cancelSubscription
POST /api/sendEmail
```

Every operation becomes a POST, every URL is a verb phrase, and the uniform interface constraint is violated. The better model:

```
GET    /api/v1/users/{id}
POST   /api/v1/orders
PATCH  /api/v1/subscriptions/{id}          (body: { "status": "cancelled" })
POST   /api/v1/emails                       (or POST /api/v1/notifications)
```

### Resource Naming Rules

**Use nouns, not verbs.** The HTTP method is the verb. The URL is the noun.

**Use plural nouns for collections.** `/users` not `/user`. `/orders` not `/order`. Consistency matters more than grammar.

**Use lowercase and hyphens for multi-word resources.** `/blog-posts`, not `/blogPosts` or `/blog_posts`. URLs are case-sensitive and lowercase is the convention.

**Nest resources to express relationships, but only one level deep.** `/users/{userId}/orders` is fine. `/users/{userId}/orders/{orderId}/items/{itemId}/reviews` is too deep — flatten it.

```
# Appropriate nesting
GET /api/v1/users/{userId}/orders          # Orders belonging to a user
GET /api/v1/orders/{orderId}/items         # Items within an order

# Too deep — flatten this
GET /api/v1/users/{userId}/orders/{orderId}/items/{itemId}
# Better:
GET /api/v1/order-items/{itemId}
```

**Actions that do not map cleanly to CRUD can use a sub-resource noun.** Some operations are inherently action-like — publishing a post, cancelling an order, verifying an email. The cleanest approach is to model the result as a sub-resource:

```
POST /api/v1/orders/{orderId}/cancellation   # Cancels the order
POST /api/v1/users/{userId}/verification     # Triggers email verification
POST /api/v1/posts/{postId}/publication      # Publishes the post
```

This keeps the URL as a noun while still using POST correctly. Stripe uses this pattern throughout their API.

---

## URL Structure

A well-structured API URL has these components:

```
https://api.example.com/api/v1/users/abc-123/orders?status=pending&limit=20
|------|---------------|-----|--|-----|---------|------|-----|-----|----|--|
scheme    host         base ver resource  id     sub    query params
```

The base path (`/api`) is optional but useful — it makes it clear this is an API endpoint, not a web page. The version (`/v1`) is discussed in depth in Section 4. Keep the structure predictable: `/{resource}/{id}/{sub-resource}`.

---

## Express Implementation: Router Structure

In Express, map your resource structure directly to your router structure. Each resource gets its own router file:

```typescript
// src/features/orders/orders.router.ts
import { Router } from 'express';
import { authenticate } from '../../shared/middleware/authenticate';
import { validate } from '../../shared/middleware/validate';
import { OrdersController } from './orders.controller';
import {
  createOrderSchema,
  updateOrderSchema,
  listOrdersSchema,
} from './orders.schema';

const router = Router();
const controller = new OrdersController();

// Collection routes
router.get('/', authenticate, validate({ query: listOrdersSchema }), controller.list);
router.post('/', authenticate, validate({ body: createOrderSchema }), controller.create);

// Member routes
router.get('/:id', authenticate, controller.getById);
router.patch('/:id', authenticate, validate({ body: updateOrderSchema }), controller.update);
router.delete('/:id', authenticate, controller.remove);

// Sub-resource routes
router.post('/:id/cancellation', authenticate, controller.cancel);
router.get('/:id/items', authenticate, controller.listItems);

export default router;
```

```typescript
// src/app.ts — register all routers under a versioned base path
import usersRouter from './features/users/users.router';
import ordersRouter from './features/orders/orders.router';
import productsRouter from './features/products/products.router';

app.use('/api/v1/users', usersRouter);
app.use('/api/v1/orders', ordersRouter);
app.use('/api/v1/products', productsRouter);
```

This structure means that `/api/v1/orders/:id/cancellation` maps to `ordersRouter` → the `/:id/cancellation` POST route. Every route in your API is discoverable by looking at the router files.

---

## Sources

- Fielding, Roy Thomas. "Architectural Styles and the Design of Network-based Software Architectures." Doctoral dissertation, University of California, Irvine, 2000. https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm
- Fielding, Roy, et al. RFC 7231: "Hypertext Transfer Protocol (HTTP/1.1): Semantics and Content." IETF, 2014. https://datatracker.ietf.org/doc/html/rfc7231
- Richardson, Leonard, and Mike Amundsen. *RESTful Web APIs*. O'Reilly Media, 2013.
- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — Chapter 2: Designing a REST API.
- Google. "Google API Design Guide." https://cloud.google.com/apis/design — Resource-oriented design principles.
- Stripe API Reference. https://stripe.com/docs/api — Industry reference for consistent resource and action modeling.