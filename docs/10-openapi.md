# 10. OpenAPI Specification

## What OpenAPI Is and Why It Matters

OpenAPI (formerly Swagger) is a specification for describing REST APIs in a machine-readable format — typically YAML or JSON. An OpenAPI document describes every endpoint: its path, HTTP method, parameters, request body, response schemas, authentication requirements, and possible error responses.

The spec is not documentation for its own sake. A valid OpenAPI document enables a toolchain: generate interactive documentation automatically, generate client SDKs in multiple languages, validate requests and responses in tests, mock servers for frontend development before the backend is built, and import the API into tools like Postman or Insomnia with one click.

The current version is OpenAPI 3.1, which aligns fully with JSON Schema. Use 3.1 for new APIs.

---

## Spec-First vs Code-First

There are two approaches to maintaining an OpenAPI spec:

**Spec-first:** Write the YAML file before writing any implementation code. The spec is the contract — frontend and backend developers work against it simultaneously. A mock server generated from the spec lets the frontend make real HTTP calls before the backend exists.

**Code-first:** Generate the spec from code annotations or decorators on your route handlers. The implementation is the source of truth; the spec is derived from it.

**The recommendation is spec-first.** Writing the spec first forces you to think about the API's interface before you are invested in an implementation. It enables parallel development. It surfaces design problems early — when fixing them is cheap. The Stripe and GitHub APIs, often cited as the best-designed APIs in the industry, are maintained spec-first.

Code-first is pragmatic when adding OpenAPI to an existing API that was built without it. In that case, generating a baseline spec from the code and then refining it manually is a reasonable starting point.

---

## A Working OpenAPI 3.1 Example

The following is a realistic partial spec for an orders API. It demonstrates the patterns used throughout this guide.

```yaml
# docs/api/openapi.yaml
openapi: 3.1.0

info:
  title: Example API
  version: 1.0.0
  description: |
    REST API for the Example application.
    
    ## Authentication
    
    Most endpoints require authentication. Include a valid JWT access token
    as a Bearer token in the `Authorization` header, or pass it via the
    `accessToken` httpOnly cookie if calling from a browser.
    
    ## Rate Limiting
    
    API requests are rate limited to 100 requests per minute per authenticated user.
    Rate limit headers are included in every response.
    
    ## Errors
    
    All error responses follow the same structure with a `status`, `code`,
    and `message` field. Validation errors include a `details` array with
    field-level information.

servers:
  - url: https://api.example.com/api/v1
    description: Production
  - url: http://localhost:3000/api/v1
    description: Local development

security:
  - BearerAuth: []

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

  schemas:
    # Reusable response wrappers
    SuccessResponse:
      type: object
      required: [status, data]
      properties:
        status:
          type: string
          enum: [success]
        data:
          type: object

    ErrorResponse:
      type: object
      required: [status, code, message]
      properties:
        status:
          type: string
          enum: [error]
        code:
          type: string
          description: Machine-readable error code
          example: NOT_FOUND
        message:
          type: string
          description: Human-readable error description
        details:
          type: array
          items:
            $ref: '#/components/schemas/ValidationErrorDetail'
        requestId:
          type: string
          format: uuid

    ValidationErrorDetail:
      type: object
      required: [field, message]
      properties:
        field:
          type: string
          example: items[0].quantity
        message:
          type: string
          example: Quantity must be a positive integer.

    # Domain schemas
    Order:
      type: object
      required: [id, userId, status, total, createdAt, updatedAt]
      properties:
        id:
          type: string
          format: uuid
        userId:
          type: string
          format: uuid
        status:
          type: string
          enum: [pending, confirmed, processing, completed, cancelled]
        total:
          type: integer
          description: Total amount in smallest currency unit (e.g., cents for USD)
          example: 150000
        note:
          type: string
          nullable: true
          maxLength: 500
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        createdAt:
          type: string
          format: date-time
        updatedAt:
          type: string
          format: date-time

    OrderItem:
      type: object
      required: [id, productId, quantity, price]
      properties:
        id:
          type: string
          format: uuid
        productId:
          type: string
          format: uuid
        quantity:
          type: integer
          minimum: 1
          maximum: 999
        price:
          type: integer
          description: Unit price at time of order, in smallest currency unit

    CreateOrderRequest:
      type: object
      required: [items, shippingAddressId]
      properties:
        items:
          type: array
          minItems: 1
          maxItems: 50
          items:
            type: object
            required: [productId, quantity]
            properties:
              productId:
                type: string
                format: uuid
              quantity:
                type: integer
                minimum: 1
                maximum: 999
        shippingAddressId:
          type: string
          format: uuid
        note:
          type: string
          maxLength: 500

    PaginationMeta:
      type: object
      required: [limit]
      properties:
        limit:
          type: integer
        nextCursor:
          type: string
          nullable: true
        prevCursor:
          type: string
          nullable: true
        total:
          type: integer
          description: Total count (only for offset-based pagination)

  responses:
    Unauthorized:
      description: Authentication required or token invalid
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: error
            code: UNAUTHORIZED
            message: Authentication required.

    Forbidden:
      description: Insufficient permissions
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: error
            code: FORBIDDEN
            message: You do not have permission to perform this action.

    NotFound:
      description: Resource not found
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: error
            code: NOT_FOUND
            message: The requested resource was not found.

    ValidationError:
      description: Request validation failed
      content:
        application/json:
          schema:
            $ref: '#/components/schemas/ErrorResponse'
          example:
            status: error
            code: VALIDATION_ERROR
            message: Request validation failed.
            details:
              - field: body.items
                message: At least one item is required.

paths:
  /orders:
    get:
      operationId: listOrders
      summary: List orders
      description: Returns a paginated list of orders for the authenticated user.
      tags: [Orders]
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, confirmed, processing, completed, cancelled]
        - name: limit
          in: query
          schema:
            type: integer
            minimum: 1
            maximum: 100
            default: 20
        - name: cursor
          in: query
          schema:
            type: string
          description: Pagination cursor from previous response meta.nextCursor
        - name: sort
          in: query
          schema:
            type: string
            example: -createdAt
          description: "Sort field. Prefix with '-' for descending. Allowed: createdAt, total"
      responses:
        '200':
          description: Paginated list of orders
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [success]
                  data:
                    type: array
                    items:
                      $ref: '#/components/schemas/Order'
                  meta:
                    $ref: '#/components/schemas/PaginationMeta'
        '401':
          $ref: '#/components/responses/Unauthorized'

    post:
      operationId: createOrder
      summary: Create an order
      description: Creates a new order for the authenticated user.
      tags: [Orders]
      parameters:
        - name: Idempotency-Key
          in: header
          required: false
          schema:
            type: string
            format: uuid
          description: |
            Optional UUID to ensure idempotent order creation.
            If provided, duplicate requests with the same key return the original response.
      requestBody:
        required: true
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created successfully
          headers:
            Location:
              description: URL of the created order
              schema:
                type: string
                example: /api/v1/orders/550e8400-e29b-41d4-a716-446655440000
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [success]
                  data:
                    $ref: '#/components/schemas/Order'
        '400':
          $ref: '#/components/responses/ValidationError'
        '401':
          $ref: '#/components/responses/Unauthorized'

  /orders/{id}:
    parameters:
      - name: id
        in: path
        required: true
        schema:
          type: string
          format: uuid
        description: Order ID

    get:
      operationId: getOrder
      summary: Get an order
      tags: [Orders]
      responses:
        '200':
          description: Order details
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [success]
                  data:
                    $ref: '#/components/schemas/Order'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'

  /orders/{id}/cancellation:
    post:
      operationId: cancelOrder
      summary: Cancel an order
      description: |
        Cancels a pending or confirmed order. Returns 409 if the order
        is already cancelled or has been shipped.
      tags: [Orders]
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: string
            format: uuid
      responses:
        '200':
          description: Order cancelled successfully
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    enum: [success]
                  data:
                    $ref: '#/components/schemas/Order'
        '401':
          $ref: '#/components/responses/Unauthorized'
        '404':
          $ref: '#/components/responses/NotFound'
        '409':
          description: Order cannot be cancelled in its current state
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/ErrorResponse'
```

---

## Tooling

**Stoplight Studio** — A free desktop and web-based GUI for editing OpenAPI specs with visual validation. The best tool for writing specs from scratch without manually editing YAML.

**Swagger UI / Redoc** — Render your OpenAPI spec as interactive documentation. Both are open source. Redoc produces cleaner output; Swagger UI has the "try it" request builder built in.

**Serving docs from your Express app:**

```typescript
import swaggerUi from 'swagger-ui-express';
import YAML from 'yamljs';
import path from 'path';

const spec = YAML.load(path.join(__dirname, '../docs/api/openapi.yaml'));

// Only serve docs in non-production, or add access control
if (env.NODE_ENV !== 'production') {
  app.use('/api-docs', swaggerUi.serve, swaggerUi.setup(spec));
}
```

**Spectral** — OpenAPI linter. Validates your spec against OpenAPI rules and lets you define custom rules for your API style guide (e.g., "all endpoints must have operationId," "all responses must have examples").

```bash
npx @stoplight/spectral-cli lint docs/api/openapi.yaml
```

---

## Sources

- OpenAPI Initiative. "OpenAPI Specification 3.1.0." https://spec.openapis.org/oas/v3.1.0
- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — Chapter 10: Documenting an API.
- Stoplight. "OpenAPI Design Guide." https://apistylebook.com/
- Spectral Documentation. https://docs.stoplight.io/docs/spectral
- Redoc Documentation. https://redocly.com/docs/redoc/