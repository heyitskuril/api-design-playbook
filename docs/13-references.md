# 13. Reference Library

All sources cited throughout this guide, organized by category.

---

## Books

| Title | Author(s) | Year | Publisher | Referenced In |
|-------|-----------|------|-----------|---------------|
| *The Design of Web APIs* | Arnaud Lauret | 2019 | Manning Publications | Sections 1, 2, 4, 5, 10 |
| *RESTful Web APIs* | Leonard Richardson & Mike Amundsen | 2013 | O'Reilly Media | Section 1 |
| *HTTP: The Definitive Guide* | David Gourley, Brian Totty, et al. | 2002 | O'Reilly Media | Sections 1, 2 |
| *Designing Data-Intensive Applications* | Martin Kleppmann | 2017 | O'Reilly Media | Sections 5, 6, 9 |
| *Release It!: Design and Deploy Production-Ready Software*, 2nd ed. | Michael T. Nygard | 2018 | Pragmatic Bookshelf | Section 3 |
| *OAuth 2 in Action* | Justin Richer & Antonio Sanso | 2017 | Manning Publications | Section 7 |
| *Unit Testing: Principles, Practices, and Patterns* | Vladimir Khorikov | 2020 | Manning Publications | Section 11 |

---

## Standards & Specifications

| Document | Organization | Year | URL | Referenced In |
|----------|-------------|------|-----|---------------|
| RFC 7231: "HTTP/1.1 Semantics and Content" | IETF | 2014 | https://datatracker.ietf.org/doc/html/rfc7231 | Sections 1, 2, 4 |
| RFC 7807: "Problem Details for HTTP APIs" | IETF | 2016 | https://datatracker.ietf.org/doc/html/rfc7807 | Section 3 |
| RFC 9457: "Problem Details for HTTP APIs" (supersedes 7807) | IETF | 2023 | https://datatracker.ietf.org/doc/html/rfc9457 | Sections 2, 3 |
| RFC 8594: "The Sunset HTTP Header Field" | IETF | 2019 | https://datatracker.ietf.org/doc/html/rfc8594 | Section 4 |
| RFC 8725: "JSON Web Token Best Current Practices" | IETF | 2020 | https://datatracker.ietf.org/doc/html/rfc8725 | Section 7 |
| OpenAPI Specification 3.1.0 | OpenAPI Initiative | 2021 | https://spec.openapis.org/oas/v3.1.0 | Section 10 |
| OWASP API Security Top 10 (2023) | OWASP | 2023 | https://owasp.org/API-Security/ | Section 12 |
| OWASP Authentication Cheat Sheet | OWASP | 2024 | https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html | Sections 7, 12 |
| OWASP REST Security Cheat Sheet | OWASP | 2024 | https://cheatsheetseries.owasp.org/cheatsheets/REST_Security_Cheat_Sheet.html | Sections 5, 7, 12 |
| OWASP Input Validation Cheat Sheet | OWASP | 2024 | https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html | Section 8 |
| OWASP Mass Assignment Cheat Sheet | OWASP | 2024 | https://cheatsheetseries.owasp.org/cheatsheets/Mass_Assignment_Cheat_Sheet.html | Section 12 |
| NIST SP 800-63B: "Digital Identity Guidelines" | NIST | 2020 | https://pages.nist.gov/800-63-3/sp800-63b.html | Section 8 |
| ISO 8601: "Date and Time Format" | ISO | 2019 | https://www.iso.org/iso-8601-date-and-time-format.html | Section 2 |
| RateLimit Header Fields for HTTP (IETF Draft) | IETF | 2022 | https://datatracker.ietf.org/doc/draft-ietf-httpapi-ratelimit-headers/ | Section 9 |

---

## Official Documentation

| Tool / Platform | URL | Referenced In |
|----------------|-----|---------------|
| Express.js Documentation | https://expressjs.com/en/4x/api.html | Sections 1, 3, 5, 9 |
| Express.js Error Handling Guide | https://expressjs.com/en/guide/error-handling.html | Section 3 |
| Zod Documentation | https://zod.dev | Sections 8, 2 |
| Helmet.js Documentation | https://helmetjs.github.io/ | Sections 5, 12 |
| express-rate-limit Documentation | https://github.com/express-rate-limit/express-rate-limit | Section 9 |
| ioredis Documentation | https://github.com/redis/ioredis | Sections 7, 9 |
| Supertest Documentation | https://github.com/ladjs/supertest | Section 11 |
| Vitest Documentation | https://vitest.dev | Section 11 |
| Stoplight Spectral Documentation | https://docs.stoplight.io/docs/spectral | Section 10 |
| Redoc Documentation | https://redocly.com/docs/redoc/ | Section 10 |
| AJV Documentation | https://ajv.js.org | Section 11 |
| PostgreSQL Documentation | https://www.postgresql.org/docs/current/ | Section 5 |
| Prisma Documentation | https://www.prisma.io/docs | Section 5 |

---

## Online Articles & Engineering Resources

| Title | Author / Organization | URL | Referenced In |
|-------|----------------------|-----|---------------|
| "Architectural Styles and the Design of Network-based Software Architectures" (Fielding Dissertation) | Roy T. Fielding | https://www.ics.uci.edu/~fielding/pubs/dissertation/top.htm | Section 1 |
| "Error Handling in Node.js" | Joyent | https://www.joyent.com/node-js/production/design/errors | Section 3 |
| "API Versioning" | Stripe Engineering Blog | https://stripe.com/blog/api-versioning | Section 4 |
| "Your API versioning is wrong" | Troy Hunt | https://www.troyhunt.com/your-api-versioning-is-wrong-which-is | Section 4 |
| "REST API Design — Versioning" | Microsoft Azure Architecture | https://learn.microsoft.com/en-us/azure/architecture/best-practices/api-design | Section 4 |
| "Evolving API Pagination at Slack" | Slack Engineering | https://slack.engineering/evolving-api-pagination-at-slack/ | Section 5 |
| "Using pagination in the REST API" | GitHub | https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api | Section 5 |
| "Idempotent Requests" | Stripe API Documentation | https://stripe.com/docs/api/idempotent_requests | Section 6 |
| "Making Retries Safe with Idempotent APIs" | AWS Builders Library | https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/ | Section 6 |
| Google API Design Guide | Google Cloud | https://cloud.google.com/apis/design | Section 1 |
| "Blocking Brute Force Attacks" | OWASP | https://owasp.org/www-community/controls/Blocking_Brute_Force_Attacks | Section 9 |
| Mozilla HTTP Observatory | Mozilla | https://observatory.mozilla.org/ | Section 12 |
| Security Headers Scanner | Scott Helme | https://securityheaders.com/ | Section 12 |
| OWASP Server Side Request Forgery Prevention | OWASP | https://cheatsheetseries.owasp.org/cheatsheets/Server_Side_Request_Forgery_Prevention_Cheat_Sheet.html | Section 12 |