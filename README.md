<div align="center">

# API Design Playbook

**A practical reference for designing, building, and documenting production-grade REST APIs with Node.js and Express.**

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](./LICENSE)
[![Contributions Welcome](https://img.shields.io/badge/contributions-welcome-brightgreen.svg)](./CONTRIBUTING.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](./CONTRIBUTING.md)
![Last Updated](https://img.shields.io/badge/last%20updated-2026-blue)

</div>

---

This is not a beginner guide to what an API is. It is a reference for developers who are already building APIs and want to do it well — consistently, defensibly, and in a way that does not create problems for the people consuming them.

Every section covers the reasoning behind a decision, not just the decision itself. Where the industry has a clear standard (OpenAPI, RFC specifications, OWASP), that standard is cited. Where there is genuine trade-off between approaches, both sides are stated.

The code examples throughout use Express and Node.js with TypeScript.

---

## Who This Is For

| Audience | How to use this |
|----------|----------------|
| Backend developers building their first production API | Read in order — each section builds on the last |
| Developers inheriting an existing API and trying to improve it | Jump to the sections covering the problems you are facing |
| Teams establishing shared API conventions | Use individual sections as the basis for internal engineering decisions |
| Frontend developers who want to understand what makes an API easy to consume | Sections 2, 4, 5, and 6 are particularly relevant |

---

## What Is Covered

| # | Section | Summary |
|---|---------|---------|
| 1 | [REST Fundamentals](./docs/01-rest-fundamentals.md) | What REST actually is, constraints, resource modeling, HTTP semantics |
| 2 | [Request & Response Design](./docs/02-request-response-design.md) | Consistent response shapes, status codes, naming conventions |
| 3 | [Error Handling](./docs/03-error-handling.md) | Error response standards, operational vs programmer errors, client-friendly messages |
| 4 | [Versioning](./docs/04-versioning.md) | URL vs header versioning, when to version, backwards compatibility |
| 5 | [Pagination, Filtering & Sorting](./docs/05-pagination-filtering-sorting.md) | Cursor vs offset pagination, filter conventions, sort parameters |
| 6 | [Idempotency](./docs/06-idempotency.md) | What idempotency is, idempotency keys, safe retries |
| 7 | [Authentication & Authorization in APIs](./docs/07-auth.md) | JWT in APIs, API keys, scopes, per-endpoint authorization |
| 8 | [Validation](./docs/08-validation.md) | Input validation with Zod, schema-first validation, sanitization |
| 9 | [Rate Limiting & Throttling](./docs/09-rate-limiting.md) | Rate limit strategies, headers, response format, Redis-backed limiting |
| 10 | [OpenAPI Specification](./docs/10-openapi.md) | Writing OpenAPI 3.1 specs, code-first vs spec-first, tooling |
| 11 | [Testing APIs](./docs/11-testing.md) | Integration testing with Supertest, contract testing, test structure |
| 12 | [Security](./docs/12-security.md) | OWASP API Security Top 10, headers, CORS, injection prevention |
| 13 | [Reference Library](./docs/13-references.md) | All books and resources cited throughout this guide |

Each section includes working Express/TypeScript code examples, the reasoning behind each decision, and explicit trade-offs.

---

## Tech Stack

![Node.js](https://img.shields.io/badge/Node.js-339933?style=flat-square&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=flat-square&logo=express&logoColor=white)
![TypeScript](https://img.shields.io/badge/TypeScript-3178C6?style=flat-square&logo=typescript&logoColor=white)
![Zod](https://img.shields.io/badge/Zod-3E67B1?style=flat-square&logo=zod&logoColor=white)
![OpenAPI](https://img.shields.io/badge/OpenAPI-6BA539?style=flat-square&logo=openapiinitiative&logoColor=white)
![Vitest](https://img.shields.io/badge/Vitest-6E9F18?style=flat-square&logo=vitest&logoColor=white)

---

## How to Use This

**If you are designing a new API from scratch,** read Sections 1 through 6 before writing any code. These cover the decisions that are expensive to change later — resource naming, response shape, versioning strategy, and error format.

**If you are improving an existing API,** each section is self-contained. You can jump to the problem you are trying to solve. Section 3 (Error Handling) and Section 5 (Pagination) are the most commonly neglected areas in APIs that have grown organically.

**If you are writing an OpenAPI spec for the first time,** read Section 2 and Section 3 first so you understand what you are documenting, then move to Section 10.

---

## Companion Repositories

This guide is part of a series on engineering and product development:

- [Product Development Playbook](https://github.com/heyitskuril/product-development-playbook) — The complete 17-phase guide from idea to launch
- [PERN Stack Architecture Guide](https://github.com/heyitskuril/pern-stack-architecture-guide) — Clean architecture, layering, and project structure for PERN apps
- [System Design for Web Developers](https://github.com/heyitskuril/system-design-for-web-developers) — A practical system design reference for developers who build real products

---

## References & Standards

This guide draws from:

- Books: *The Design of Web APIs* (Lauret), *RESTful Web APIs* (Richardson & Amundsen), *HTTP: The Definitive Guide* (Gourley et al.), and others — all cited inline
- Standards: OWASP API Security Top 10, OpenAPI 3.1 Specification, RFC 7231 (HTTP Semantics), RFC 7807 (Problem Details), RFC 9457 (Problem Details — updated)
- Engineering resources: Stripe API documentation, GitHub API documentation, Google API Design Guide

The full list is in [Section 13 — Reference Library](./docs/13-references.md).

---

## Contributing

See [CONTRIBUTING.md](./CONTRIBUTING.md) for guidelines on fixing errors, adding references, or suggesting new sections.

---

## License

MIT License — free to use, adapt, and distribute. See [LICENSE](./LICENSE).

---

## About the author

<div align="center">

<img src="https://avatars.githubusercontent.com/heyitskuril" width="80" style="border-radius: 50%;" />

**Kuril**

A dreamer… who still working on it to make it real.

[![LinkedIn](https://img.shields.io/badge/LinkedIn-0A66C2?style=flat-square&logo=linkedin&logoColor=white)](https://linkedin.com/in/heyitskuril)
[![Instagram](https://img.shields.io/badge/Instagram-E4405F?style=flat-square&logo=instagram&logoColor=white)](https://instagram.com/heyitskuril)
[![GitHub](https://img.shields.io/badge/GitHub-181717?style=flat-square&logo=github&logoColor=white)](https://github.com/heyitskuril)
[![Email](https://img.shields.io/badge/Email-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:kuril.dev@gmail.com)

</div>

---

<div align="center">

If this guide helped you, consider giving it a ⭐ — it helps others find it too.

</div>
