# REST API Best Practices

## URL Structure

### Resource Naming

```
# Good - Plural nouns
GET /api/users
GET /api/orders
GET /api/products

# Bad - Verbs or mixed conventions
GET /api/getUser
GET /api/user  (inconsistent singular)
POST /api/createOrder
```

### Nested Resources

```
# Shallow nesting (preferred)
GET /api/users/:id/orders
GET /api/orders/:id

# Deep nesting (avoid)
GET /api/users/:id/orders/:orderId/items/:itemId/reviews
# Better:
GET /api/order-items/:id/reviews
```

## HTTP Methods and Status Codes

### GET - Retrieve Resources

```
GET /api/users              → 200 OK (with list)
GET /api/users/:id          → 200 OK or 404 Not Found
GET /api/users?page=2       → 200 OK (paginated)
```

### POST - Create Resources

```
POST /api/users
  Body: {"name": "John", "email": "john@example.com"}
  → 201 Created
  Location: /api/users/123
  Body: {"id": "123", "name": "John", ...}

POST /api/users (validation error)
  → 422 Unprocessable Entity
  Body: {"errors": [...]}
```

### PUT - Replace Resources

```
PUT /api/users/:id
  Body: {complete user object}
  → 200 OK (updated)
  → 404 Not Found (doesn't exist)

# Must include ALL fields
```

### PATCH - Partial Update

```
PATCH /api/users/:id
  Body: {"name": "Jane"}  (only changed fields)
  → 200 OK
  → 404 Not Found
```

### DELETE - Remove Resources

```
DELETE /api/users/:id
  → 204 No Content (deleted)
  → 404 Not Found
  → 409 Conflict (can't delete due to references)
```

## Filtering, Sorting, and Searching

### Query Parameters

```
# Filtering
GET /api/users?status=active
GET /api/users?role=admin&status=active

# Sorting
GET /api/users?sort=created_at
GET /api/users?sort=-created_at  (descending)
GET /api/users?sort=name,created_at

# Searching
GET /api/users?search=john
GET /api/users?q=john

# Field selection (sparse fieldsets)
GET /api/users?fields=id,name,email
```

## Pagination Patterns

### Offset-Based Pagination

```typescript
GET /api/users?page=2&pageSize=20

Response:
{
  "items": [...],
  "page": 2,
  "pageSize": 20,
  "total": 150,
  "pages": 8,
  "hasNext": true,
  "hasPrev": true
}
```

### Cursor-Based Pagination (for large datasets)

```typescript
GET /api/users?limit=20&cursor=eyJpZCI6MTIzfQ

Response:
{
  "items": [...],
  "nextCursor": "eyJpZCI6MTQzfQ",
  "hasMore": true
}
```

### Link Header Pagination (RESTful)

```
GET /api/users?page=2

Response Headers:
Link: <https://api.example.com/users?page=3>; rel="next",
      <https://api.example.com/users?page=1>; rel="prev",
      <https://api.example.com/users?page=1>; rel="first",
      <https://api.example.com/users?page=8>; rel="last"
```

## Versioning Strategies

### URL Versioning (Recommended)

```
/api/v1/users
/api/v2/users

Pros: Clear, easy to route
Cons: Multiple URLs for same resource
```

### Header Versioning

```
GET /api/users
Accept: application/vnd.api+json; version=2

Pros: Clean URLs
Cons: Less visible, harder to test
```

### Query Parameter

```
GET /api/users?version=2

Pros: Easy to test
Cons: Optional parameter can be forgotten
```

## Rate Limiting

### Headers

```
X-RateLimit-Limit: 1000
X-RateLimit-Remaining: 742
X-RateLimit-Reset: 1640000000

Response when limited:
429 Too Many Requests
Retry-After: 3600
```

### Implementation Pattern

```typescript
import { Request, Response, NextFunction } from "express";

interface RateLimitEntry {
  timestamps: Date[];
}

class RateLimiter {
  private cache: Map<string, RateLimitEntry>;

  constructor(
    private readonly calls: number,
    private readonly period: number // seconds
  ) {
    this.cache = new Map();
  }

  check(key: string): boolean {
    const now = new Date();
    const entry = this.cache.get(key) || { timestamps: [] };

    // Remove expired timestamps
    entry.timestamps = entry.timestamps.filter(
      (ts) => now.getTime() - ts.getTime() < this.period * 1000
    );

    if (entry.timestamps.length >= this.calls) {
      return false;
    }

    entry.timestamps.push(now);
    this.cache.set(key, entry);
    return true;
  }

  getRemaining(key: string): number {
    const entry = this.cache.get(key);
    if (!entry) return this.calls;

    const now = new Date();
    const validTimestamps = entry.timestamps.filter(
      (ts) => now.getTime() - ts.getTime() < this.period * 1000
    );

    return Math.max(0, this.calls - validTimestamps.length);
  }

  getResetTime(key: string): number {
    const entry = this.cache.get(key);
    if (!entry || entry.timestamps.length === 0) {
      return Math.floor(Date.now() / 1000) + this.period;
    }

    const oldestTimestamp = entry.timestamps[0];
    return Math.floor(oldestTimestamp.getTime() / 1000) + this.period;
  }
}

const limiter = new RateLimiter(100, 60); // 100 calls per 60 seconds

function rateLimitMiddleware(req: Request, res: Response, next: NextFunction) {
  const key = req.ip || req.socket.remoteAddress || "unknown";

  if (!limiter.check(key)) {
    res.setHeader("X-RateLimit-Limit", "100");
    res.setHeader("X-RateLimit-Remaining", "0");
    res.setHeader("X-RateLimit-Reset", limiter.getResetTime(key).toString());
    res.setHeader("Retry-After", "60");

    return res.status(429).json({
      error: "TooManyRequests",
      message: "Rate limit exceeded",
      retryAfter: 60,
    });
  }

  res.setHeader("X-RateLimit-Limit", "100");
  res.setHeader("X-RateLimit-Remaining", limiter.getRemaining(key).toString());
  res.setHeader("X-RateLimit-Reset", limiter.getResetTime(key).toString());

  next();
}

// Usage
import express from "express";

const app = express();

app.use("/api", rateLimitMiddleware);

app.get("/api/users", (req, res) => {
  res.json({ users: [] });
});
```

## Authentication and Authorization

### Bearer Token

```
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...

401 Unauthorized - Missing/invalid token
403 Forbidden - Valid token, insufficient permissions
```

### API Keys

```
X-API-Key: your-api-key-here
```

## Error Response Format

### Consistent Structure

```json
{
  "error": {
    "code": "VALIDATION_ERROR",
    "message": "Request validation failed",
    "details": [
      {
        "field": "email",
        "message": "Invalid email format",
        "value": "not-an-email"
      }
    ],
    "timestamp": "2025-10-16T12:00:00Z",
    "path": "/api/users"
  }
}
```

### Status Code Guidelines

- `200 OK`: Successful GET, PATCH, PUT
- `201 Created`: Successful POST
- `204 No Content`: Successful DELETE
- `400 Bad Request`: Malformed request
- `401 Unauthorized`: Authentication required
- `403 Forbidden`: Authenticated but not authorized
- `404 Not Found`: Resource doesn't exist
- `409 Conflict`: State conflict (duplicate email, etc.)
- `422 Unprocessable Entity`: Validation errors
- `429 Too Many Requests`: Rate limited
- `500 Internal Server Error`: Server error
- `503 Service Unavailable`: Temporary downtime

## Caching

### Cache Headers

```
# Client caching
Cache-Control: public, max-age=3600

# No caching
Cache-Control: no-cache, no-store, must-revalidate

# Conditional requests
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
→ 304 Not Modified
```

## Bulk Operations

### Batch Endpoints

```typescript
POST /api/users/batch
{
  "items": [
    {"name": "User1", "email": "user1@example.com"},
    {"name": "User2", "email": "user2@example.com"}
  ]
}

Response:
{
  "results": [
    {"id": "1", "status": "created"},
    {"id": null, "status": "failed", "error": "Email already exists"}
  ]
}

// Implementation
import { Request, Response } from 'express';

interface BatchCreateRequest {
  items: Array<{ name: string; email: string }>;
}

interface BatchResult {
  id: string | null;
  status: 'created' | 'failed';
  error?: string;
}

router.post('/api/users/batch', async (req: Request, res: Response) => {
  const { items } = req.body as BatchCreateRequest;

  const results: BatchResult[] = await Promise.all(
    items.map(async (item) => {
      try {
        const user = await createUser(item);
        return { id: user.id, status: 'created' as const };
      } catch (error) {
        return {
          id: null,
          status: 'failed' as const,
          error: error instanceof Error ? error.message : 'Unknown error'
        };
      }
    })
  );

  res.json({ results });
});
```

## Idempotency

### Idempotency Keys

```typescript
POST /api/orders
Idempotency-Key: unique-key-123

If duplicate request:
→ 200 OK (return cached response)

// Implementation
import { Request, Response, NextFunction } from 'express';

interface IdempotencyCache {
  [key: string]: {
    statusCode: number;
    body: any;
    timestamp: Date;
  };
}

const idempotencyCache: IdempotencyCache = {};
const CACHE_TTL = 24 * 60 * 60 * 1000; // 24 hours

function idempotencyMiddleware(req: Request, res: Response, next: NextFunction) {
  const idempotencyKey = req.get('Idempotency-Key');

  if (!idempotencyKey) {
    return next();
  }

  // Check cache
  const cached = idempotencyCache[idempotencyKey];
  if (cached) {
    const age = Date.now() - cached.timestamp.getTime();
    if (age < CACHE_TTL) {
      return res.status(cached.statusCode).json(cached.body);
    }
    // Cache expired
    delete idempotencyCache[idempotencyKey];
  }

  // Intercept response
  const originalJson = res.json.bind(res);
  res.json = function(body: any) {
    idempotencyCache[idempotencyKey] = {
      statusCode: res.statusCode,
      body,
      timestamp: new Date()
    };
    return originalJson(body);
  };

  next();
}

// Usage
app.post('/api/orders', idempotencyMiddleware, async (req, res) => {
  const order = await createOrder(req.body);
  res.status(201).json(order);
});
```

## CORS Configuration

```typescript
import express from "express";
import cors from "cors";

const app = express();

// Basic CORS
app.use(
  cors({
    origin: "https://example.com",
    credentials: true,
    methods: ["GET", "POST", "PUT", "PATCH", "DELETE"],
    allowedHeaders: ["Content-Type", "Authorization"],
  })
);

// Dynamic CORS
const allowedOrigins = ["https://example.com", "https://app.example.com"];

app.use(
  cors({
    origin: (origin, callback) => {
      if (!origin || allowedOrigins.includes(origin)) {
        callback(null, true);
      } else {
        callback(new Error("Not allowed by CORS"));
      }
    },
    credentials: true,
  })
);
```

## Documentation with OpenAPI

```typescript
import express from "express";
import swaggerJsdoc from "swagger-jsdoc";
import swaggerUi from "swagger-ui-express";

const app = express();

const swaggerOptions = {
  definition: {
    openapi: "3.0.0",
    info: {
      title: "My API",
      version: "1.0.0",
      description: "API for managing users",
    },
    servers: [
      {
        url: "http://localhost:3000",
        description: "Development server",
      },
    ],
  },
  apis: ["./routes/*.ts"], // Path to API docs
};

const swaggerSpec = swaggerJsdoc(swaggerOptions);

app.use("/docs", swaggerUi.serve, swaggerUi.setup(swaggerSpec));
app.get("/docs.json", (req, res) => {
  res.json(swaggerSpec);
});

/**
 * @openapi
 * /api/users/{userId}:
 *   get:
 *     summary: Get user by ID
 *     description: Retrieve user by ID. Returns full user profile including basic information, contact details, and account status.
 *     tags:
 *       - Users
 *     parameters:
 *       - in: path
 *         name: userId
 *         required: true
 *         schema:
 *           type: string
 *         description: The user ID
 *     responses:
 *       200:
 *         description: User details
 *         content:
 *           application/json:
 *             schema:
 *               type: object
 *               properties:
 *                 id:
 *                   type: string
 *                 name:
 *                   type: string
 *                 email:
 *                   type: string
 *       404:
 *         description: User not found
 */

app.get("/api/users/:userId", async (req, res) => {
  const user = await getUser(req.params.userId);
  if (!user) {
    return res.status(404).json({ error: "User not found" });
  }
  res.json(user);
});
```
