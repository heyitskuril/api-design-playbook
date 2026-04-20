# 8. Validation

## Where Validation Belongs

Validation happens in two places, for two different reasons:

**Client-side validation** is for user experience. It catches errors before a network request is made, giving instant feedback. It reduces unnecessary traffic. It is also completely bypassable — anyone with a REST client can skip your frontend entirely and send malformed requests directly to your API. Client-side validation is not security.

**Server-side validation is security.** Every request reaching your API must be validated on the server, unconditionally, regardless of what the client claims to have validated. This is not optional.

Within the server, validation belongs at the boundary — in the router layer, before the request reaches the service layer. The service layer should be able to assume that its inputs are valid and focus on business logic. Validating in the service layer means validation is scattered and inconsistently applied.

---

## Zod as the Validation Layer

Zod is the standard choice for TypeScript applications. It defines schemas that validate at runtime and simultaneously infer TypeScript types, which means your validated data is also correctly typed — you do not maintain the type and the validation separately.

```typescript
// features/orders/orders.schema.ts
import { z } from 'zod';

export const createOrderSchema = z.object({
  items: z
    .array(
      z.object({
        productId: z.string().uuid('Product ID must be a valid UUID.'),
        quantity: z
          .number()
          .int('Quantity must be a whole number.')
          .min(1, 'Quantity must be at least 1.')
          .max(999, 'Quantity cannot exceed 999.'),
        // Price is set server-side — do not trust client-supplied prices
      }),
    )
    .min(1, 'An order must contain at least one item.')
    .max(50, 'An order cannot contain more than 50 items.'),
  shippingAddressId: z.string().uuid('Shipping address ID must be a valid UUID.'),
  note: z.string().max(500, 'Note cannot exceed 500 characters.').optional(),
});

export const updateOrderSchema = z.object({
  note: z.string().max(500).optional(),
  // Status is changed through explicit sub-resource endpoints, not PATCH
});

export const listOrdersSchema = z.object({
  status: z
    .enum(['pending', 'confirmed', 'processing', 'completed', 'cancelled'])
    .optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
  sort: z.string().optional(),
});

// Type inference — TypeScript type is derived automatically
export type CreateOrderDto = z.infer<typeof createOrderSchema>;
export type ListOrdersQuery = z.infer<typeof listOrdersSchema>;
```

---

## Validation Middleware

A single validation middleware that can validate the request body, query parameters, and URL parameters:

```typescript
// shared/middleware/validate.ts
import { Request, Response, NextFunction } from 'express';
import { ZodSchema, ZodError } from 'zod';

interface ValidationTargets {
  body?: ZodSchema;
  query?: ZodSchema;
  params?: ZodSchema;
}

export function validate(schemas: ValidationTargets) {
  return (req: Request, res: Response, next: NextFunction): void => {
    const errors: Array<{ field: string; message: string }> = [];

    // Validate body
    if (schemas.body) {
      const result = schemas.body.safeParse(req.body);
      if (!result.success) {
        errors.push(...formatZodErrors(result.error, 'body'));
      } else {
        req.body = result.data; // Replace with parsed/transformed data
      }
    }

    // Validate query parameters
    if (schemas.query) {
      const result = schemas.query.safeParse(req.query);
      if (!result.success) {
        errors.push(...formatZodErrors(result.error, 'query'));
      } else {
        req.query = result.data as typeof req.query;
      }
    }

    // Validate URL parameters
    if (schemas.params) {
      const result = schemas.params.safeParse(req.params);
      if (!result.success) {
        errors.push(...formatZodErrors(result.error, 'params'));
      } else {
        req.params = result.data as typeof req.params;
      }
    }

    if (errors.length > 0) {
      res.status(400).json({
        status: 'error',
        code: 'VALIDATION_ERROR',
        message: 'Request validation failed.',
        details: errors,
        requestId: req.requestId,
      });
      return;
    }

    next();
  };
}

function formatZodErrors(
  error: ZodError,
  prefix: string,
): Array<{ field: string; message: string }> {
  return error.errors.map((e) => ({
    field: [prefix, ...e.path].join('.'),
    message: e.message,
  }));
}
```

Usage in a router:

```typescript
// features/orders/orders.router.ts
router.post(
  '/',
  authenticate,
  validate({ body: createOrderSchema }),
  asyncHandler(controller.create),
);

router.get(
  '/',
  authenticate,
  validate({ query: listOrdersSchema }),
  asyncHandler(controller.list),
);

// Validate both params and body
router.patch(
  '/:id',
  authenticate,
  validate({
    params: z.object({ id: z.string().uuid('Invalid order ID.') }),
    body: updateOrderSchema,
  }),
  asyncHandler(controller.update),
);
```

---

## Common Validation Patterns

### Required vs Optional Fields

In Zod, all fields are required by default. Use `.optional()` for fields that may be absent and `.nullable()` for fields that may be `null`. They are different:

```typescript
z.object({
  name: z.string(),           // Required, cannot be null
  bio: z.string().optional(), // May be absent (undefined)
  website: z.string().nullable(), // May be null but must be present
  phone: z.string().optional().nullable(), // May be absent or null
});
```

### String Validation

```typescript
// Email
email: z.string().email('Please provide a valid email address.').toLowerCase(),

// Password
password: z
  .string()
  .min(8, 'Password must be at least 8 characters.')
  .max(128, 'Password cannot exceed 128 characters.'),

// URL
website: z.string().url('Please provide a valid URL.').optional(),

// UUID
id: z.string().uuid('Invalid ID format.'),

// Trimming and normalization
name: z.string().min(1).max(100).trim(),

// Enum
status: z.enum(['active', 'inactive', 'suspended']),
```

### Number Validation

```typescript
// Integer with bounds
quantity: z.number().int().min(1).max(1000),

// Positive float
price: z.number().positive('Price must be a positive number.'),

// Coerce from string (for query parameters, which are always strings)
page: z.coerce.number().int().min(1).default(1),
```

### Date Validation

```typescript
// ISO 8601 string
startDate: z.string().datetime({ offset: true, message: 'Invalid date format. Use ISO 8601.' }),

// Transform string to Date object
startDate: z.string().datetime().transform((val) => new Date(val)),

// Date range validation
const dateRangeSchema = z
  .object({
    startDate: z.string().datetime().transform((val) => new Date(val)),
    endDate: z.string().datetime().transform((val) => new Date(val)),
  })
  .refine(
    (data) => data.endDate > data.startDate,
    { message: 'End date must be after start date.', path: ['endDate'] },
  );
```

### Cross-Field Validation

Use `.refine()` for rules that span multiple fields:

```typescript
const changePasswordSchema = z
  .object({
    currentPassword: z.string().min(1, 'Current password is required.'),
    newPassword: z.string().min(8, 'New password must be at least 8 characters.'),
    confirmPassword: z.string(),
  })
  .refine((data) => data.newPassword === data.confirmPassword, {
    message: 'Passwords do not match.',
    path: ['confirmPassword'],
  })
  .refine((data) => data.newPassword !== data.currentPassword, {
    message: 'New password must be different from the current password.',
    path: ['newPassword'],
  });
```

---

## What Not to Validate in Middleware

Validation middleware checks the shape and format of a request. Business constraints — "this productId must exist in the database," "this user's account must be active before they can place an order" — are not format validations. They belong in the service layer, where they throw `AppError` instances that the global error handler converts to appropriate HTTP responses.

```typescript
// In validation middleware — correct
// Check that the field is a valid UUID format
productId: z.string().uuid('Product ID must be a valid UUID.')

// In service layer — correct
// Check that the product actually exists
const product = await this.productsRepository.findById(dto.productId);
if (!product) {
  throw new AppError('NOT_FOUND', `Product ${dto.productId} not found.`, 404);
}

// Never do this in validation middleware
productId: z.string().uuid().refine(
  async (id) => !!(await prisma.product.findUnique({ where: { id } })),
  'Product not found.'
) // Async DB calls in Zod refine inside middleware is an antipattern
```

---

## Sanitization

Validation confirms that input matches the expected shape. Sanitization removes dangerous content. Both are necessary.

For text that will be stored and later rendered as HTML, sanitize before storing:

```typescript
import DOMPurify from 'isomorphic-dompurify';

// For fields that accept user-provided HTML (e.g., rich text)
content: z
  .string()
  .transform((val) => DOMPurify.sanitize(val, { ALLOWED_TAGS: ['b', 'i', 'p', 'br', 'ul', 'li'] })),

// For plain text fields, strip HTML entirely
name: z
  .string()
  .transform((val) => val.replace(/<[^>]*>/g, '').trim()),
```

For most fields — names, emails, descriptions — simply stripping leading/trailing whitespace with `.trim()` is sufficient. Never insert user-provided strings directly into SQL queries — use parameterized queries (which Prisma does by default).

---

## Sources

- Zod Documentation. https://zod.dev
- OWASP. "Input Validation Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Input_Validation_Cheat_Sheet.html
- OWASP. "Cross Site Scripting Prevention Cheat Sheet." https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html
- NIST. SP 800-63B: "Digital Identity Guidelines." https://pages.nist.gov/800-63-3/sp800-63b.html — Password length and composition guidance.