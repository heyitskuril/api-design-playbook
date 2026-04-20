# 5. Pagination, Filtering & Sorting

## Why Unbounded Queries Are Not an Option

An endpoint that returns all records without a limit is not a production endpoint. It is a liability. On a small dataset it works fine, which is why it often goes unnoticed until the dataset grows. Then it causes slow responses, memory pressure on the server, and timeouts for clients.

Every endpoint that returns a collection must be paginated. This is not optional.

---

## Cursor-Based Pagination vs Offset-Based Pagination

These are the two mainstream approaches, and they have different trade-offs worth understanding before choosing.

### Offset Pagination

```
GET /api/v1/orders?page=3&limit=20
GET /api/v1/orders?offset=40&limit=20
```

Offset pagination works by skipping a number of records and taking the next `limit` records. It maps directly to SQL: `LIMIT 20 OFFSET 40`.

**The problem with offset pagination** is that it produces inconsistent results on data that changes between requests. If records are inserted or deleted while a client is paginating through pages, records may be skipped or duplicated. Consider: a client requests page 1 (records 1–20), then a new record is inserted at position 5, then the client requests page 2 (records 21–40). What was record 21 is now record 22, and record 20 from page 1 is now returned again as the first record of page 2.

Additionally, `OFFSET` in PostgreSQL performs poorly at high offsets. To return `OFFSET 100000 LIMIT 20`, the database must read and discard 100,000 rows before returning the 20 you want. At scale, this becomes a significant query cost.

**When offset is acceptable:** paginating through static or slowly-changing data where consistency between pages is not critical. Admin dashboards, report exports, and similar use cases where you know the dataset will not change mid-pagination.

### Cursor-Based Pagination

```
GET /api/v1/orders?limit=20
→ Returns records 1–20, nextCursor: "eyJpZCI6IjIwIn0="

GET /api/v1/orders?limit=20&cursor=eyJpZCI6IjIwIn0=
→ Returns records 21–40, nextCursor: "eyJpZCI6IjQwIn0="
```

Cursor pagination uses a pointer to the last seen record rather than a numeric offset. The cursor encodes enough information to fetch the next page efficiently — typically the value of the sort column(s) for the last record in the current page.

The cursor is opaque to the client (base64-encoded JSON works well) and returned in the response metadata. The client passes it back verbatim to get the next page.

**The advantages:** consistent results even when records are inserted or deleted. Efficient queries regardless of page depth — the database uses an index range scan instead of scanning and skipping. Cursors invalidate gracefully: a cursor pointing to a deleted record simply returns the next available record.

**The disadvantage:** no random page access. A client cannot jump directly to page 7. Pagination is strictly sequential: first, next, previous. This is fine for most feed-style UIs but not for paginated table views where users want to jump to a specific page.

---

## Implementation in Express

### Cursor Pagination (recommended)

```typescript
// shared/utils/pagination.ts
import { Buffer } from 'buffer';

interface CursorPayload {
  id: string;
  createdAt: string;
}

export function encodeCursor(payload: CursorPayload): string {
  return Buffer.from(JSON.stringify(payload)).toString('base64url');
}

export function decodeCursor(cursor: string): CursorPayload | null {
  try {
    return JSON.parse(Buffer.from(cursor, 'base64url').toString('utf-8')) as CursorPayload;
  } catch {
    return null;
  }
}

export interface PaginationMeta {
  limit: number;
  nextCursor: string | null;
  prevCursor: string | null;
}

export function buildPaginationMeta<T extends { id: string; createdAt: Date }>(
  items: T[],
  limit: number,
  requestedCursor?: string,
): PaginationMeta {
  const hasMore = items.length === limit;
  const lastItem = items[items.length - 1];

  return {
    limit,
    nextCursor: hasMore && lastItem
      ? encodeCursor({ id: lastItem.id, createdAt: lastItem.createdAt.toISOString() })
      : null,
    prevCursor: requestedCursor ?? null,
  };
}
```

```typescript
// features/orders/orders.repository.ts
import { prisma } from '../../infrastructure/database/prisma';
import { decodeCursor } from '../../shared/utils/pagination';

export async function findOrders(
  userId: string,
  limit: number,
  cursor?: string,
): Promise<Order[]> {
  const decodedCursor = cursor ? decodeCursor(cursor) : null;

  return prisma.order.findMany({
    where: {
      userId,
      deletedAt: null,
      // If a cursor exists, fetch records after it
      ...(decodedCursor
        ? {
            OR: [
              {
                createdAt: { lt: new Date(decodedCursor.createdAt) },
              },
              {
                createdAt: { equals: new Date(decodedCursor.createdAt) },
                id: { gt: decodedCursor.id },
              },
            ],
          }
        : {}),
    },
    orderBy: [{ createdAt: 'desc' }, { id: 'asc' }],
    take: limit,
    include: { items: true },
  });
}
```

```typescript
// features/orders/orders.controller.ts
import { buildPaginationMeta } from '../../shared/utils/pagination';
import { sendCollection } from '../../shared/utils/response';

list = asyncHandler(async (req, res) => {
  const limit = Math.min(Number(req.query.limit) || 20, 100); // Cap at 100
  const cursor = req.query.cursor as string | undefined;

  const orders = await service.listOrders(req.user.id, limit, cursor);
  const meta = buildPaginationMeta(orders, limit, cursor);

  sendCollection(res, orders, meta);
});
```

Response:

```json
{
  "status": "success",
  "data": [...],
  "meta": {
    "limit": 20,
    "nextCursor": "eyJpZCI6ImFiYzEyMyIsImNyZWF0ZWRBdCI6IjIwMjUtMDMtMTVUMDg6MzA6MDBaIn0=",
    "prevCursor": null
  }
}
```

### Offset Pagination (when appropriate)

```typescript
// For use cases where offset is acceptable
export async function findOrdersOffset(
  userId: string,
  page: number,
  limit: number,
): Promise<{ data: Order[]; total: number }> {
  const offset = (page - 1) * limit;

  const [data, total] = await Promise.all([
    prisma.order.findMany({
      where: { userId, deletedAt: null },
      orderBy: { createdAt: 'desc' },
      skip: offset,
      take: limit,
    }),
    prisma.order.count({ where: { userId, deletedAt: null } }),
  ]);

  return { data, total };
}
```

Response with offset pagination:

```json
{
  "status": "success",
  "data": [...],
  "meta": {
    "page": 2,
    "limit": 20,
    "total": 157,
    "totalPages": 8
  }
}
```

---

## Filtering

Filtering lets clients narrow a collection to records matching specific criteria. Use query parameters for filters.

```
GET /api/v1/orders?status=pending
GET /api/v1/orders?status=pending&status=confirmed     (multi-value)
GET /api/v1/orders?createdAfter=2025-01-01&createdBefore=2025-03-31
GET /api/v1/products?minPrice=10000&maxPrice=500000&category=electronics
```

Validate filter parameters with Zod before they reach the repository:

```typescript
// features/orders/orders.schema.ts
import { z } from 'zod';

export const listOrdersSchema = z.object({
  status: z
    .union([
      z.enum(['pending', 'confirmed', 'processing', 'completed', 'cancelled']),
      z.array(z.enum(['pending', 'confirmed', 'processing', 'completed', 'cancelled'])),
    ])
    .optional()
    .transform((val) => (val ? (Array.isArray(val) ? val : [val]) : undefined)),
  createdAfter: z.string().datetime({ offset: true }).optional(),
  createdBefore: z.string().datetime({ offset: true }).optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
});

export type ListOrdersQuery = z.infer<typeof listOrdersSchema>;
```

In the repository, translate filter parameters to Prisma `where` conditions:

```typescript
export async function findOrdersWithFilters(
  userId: string,
  filters: ListOrdersQuery,
): Promise<Order[]> {
  return prisma.order.findMany({
    where: {
      userId,
      deletedAt: null,
      ...(filters.status ? { status: { in: filters.status } } : {}),
      ...(filters.createdAfter || filters.createdBefore
        ? {
            createdAt: {
              ...(filters.createdAfter ? { gte: new Date(filters.createdAfter) } : {}),
              ...(filters.createdBefore ? { lte: new Date(filters.createdBefore) } : {}),
            },
          }
        : {}),
    },
    orderBy: { createdAt: 'desc' },
    take: filters.limit,
  });
}
```

---

## Sorting

Allow clients to sort collections on specified fields. Use a `sort` or `orderBy` query parameter with a consistent format. Two common conventions:

**Convention A: separate field and direction parameters**
```
GET /api/v1/orders?sortBy=createdAt&sortDir=desc
GET /api/v1/orders?sortBy=total&sortDir=asc
```

**Convention B: prefixed field name (GitHub uses this)**
```
GET /api/v1/orders?sort=-createdAt    (minus = descending)
GET /api/v1/orders?sort=total         (no prefix = ascending)
GET /api/v1/orders?sort=-createdAt,total   (multi-sort)
```

Convention B is more compact and handles multi-column sorting cleanly. Choose one and apply it everywhere.

Always define a whitelist of sortable fields. Never pass client-supplied field names directly to the database — this is a potential injection vector.

```typescript
const SORTABLE_FIELDS = ['createdAt', 'total', 'status'] as const;
type SortableField = typeof SORTABLE_FIELDS[number];

function parseSortParam(sort: string): { field: SortableField; direction: 'asc' | 'desc' }[] {
  return sort
    .split(',')
    .map((s) => s.trim())
    .filter(Boolean)
    .map((s) => {
      const descending = s.startsWith('-');
      const field = (descending ? s.slice(1) : s) as SortableField;

      if (!SORTABLE_FIELDS.includes(field)) {
        throw new AppError('INVALID_INPUT', `Cannot sort by field: ${field}`, 400);
      }

      return { field, direction: descending ? 'desc' : 'asc' };
    });
}
```

---

## Sources

- Kleppmann, Martin. *Designing Data-Intensive Applications*. O'Reilly Media, 2017. — Chapter 3: Storage and Retrieval — index performance characteristics relevant to pagination strategies.
- Slack Engineering. "Evolving API Pagination at Slack." https://slack.engineering/evolving-api-pagination-at-slack/ — Real-world migration from offset to cursor pagination.
- GitHub REST API Documentation. "Using pagination in the REST API." https://docs.github.com/en/rest/using-the-rest-api/using-pagination-in-the-rest-api
- Lauret, Arnaud. *The Design of Web APIs*. Manning Publications, 2019. — Chapter 5: Designing a response.
- PostgreSQL Documentation. "LIMIT and OFFSET." https://www.postgresql.org/docs/current/queries-limit.html