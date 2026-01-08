# API Design Principles

Master REST and GraphQL API design principles to build intuitive, scalable, and maintainable APIs that delight developers and stand the test of time.

## When to Use This Skill

- Designing new REST or GraphQL APIs
- Refactoring existing APIs for better usability
- Establishing API design standards for your team
- Reviewing API specifications before implementation
- Migrating between API paradigms (REST to GraphQL, etc.)
- Creating developer-friendly API documentation
- Optimizing APIs for specific use cases (mobile, third-party integrations)

## Core Concepts

### 1. RESTful Design Principles

**Resource-Oriented Architecture**

- Resources are nouns (users, orders, products), not verbs
- Use HTTP methods for actions (GET, POST, PUT, PATCH, DELETE)
- URLs represent resource hierarchies
- Consistent naming conventions

**HTTP Methods Semantics:**

- `GET`: Retrieve resources (idempotent, safe)
- `POST`: Create new resources
- `PUT`: Replace entire resource (idempotent)
- `PATCH`: Partial resource updates
- `DELETE`: Remove resources (idempotent)

### 2. GraphQL Design Principles

**Schema-First Development**

- Types define your domain model
- Queries for reading data
- Mutations for modifying data
- Subscriptions for real-time updates

**Query Structure:**

- Clients request exactly what they need
- Single endpoint, multiple operations
- Strongly typed schema
- Introspection built-in

### 3. API Versioning Strategies

**URL Versioning:**

```
/api/v1/users
/api/v2/users
```

**Header Versioning:**

```
Accept: application/vnd.api+json; version=1
```

**Query Parameter Versioning:**

```
/api/users?version=1
```

## REST API Design Patterns

### Pattern 1: Resource Collection Design

```typescript
// Good: Resource-oriented endpoints
GET    /api/users              // List users (with pagination)
POST   /api/users              // Create user
GET    /api/users/:id          // Get specific user
PUT    /api/users/:id          // Replace user
PATCH  /api/users/:id          // Update user fields
DELETE /api/users/:id          // Delete user

// Nested resources
GET    /api/users/:id/orders   // Get user's orders
POST   /api/users/:id/orders   // Create order for user

// Bad: Action-oriented endpoints (avoid)
POST   /api/createUser
POST   /api/getUserById
POST   /api/deleteUser
```

### Pattern 2: Pagination and Filtering

```typescript
import { Request, Response } from "express";
import { z } from "zod";

// Validation schemas
const PaginationSchema = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(100).default(20),
});

const FilterSchema = z.object({
  status: z.string().optional(),
  createdAfter: z.string().datetime().optional(),
  search: z.string().optional(),
});

// Response types
interface PaginatedResponse<T> {
  items: T[];
  total: number;
  page: number;
  pageSize: number;
  pages: number;
  hasNext: boolean;
  hasPrev: boolean;
}

import express from "express";

const router = express.Router();

router.get("/api/users", async (req: Request, res: Response) => {
  try {
    const pagination = PaginationSchema.parse(req.query);
    const filters = FilterSchema.parse(req.query);

    const query = buildQuery(filters);
    const total = await countUsers(query);

    const offset = (pagination.page - 1) * pagination.pageSize;
    const users = await fetchUsers(query, {
      limit: pagination.pageSize,
      offset,
    });
    const pages = Math.ceil(total / pagination.pageSize);

    const response: PaginatedResponse<User> = {
      items: users,
      total,
      page: pagination.page,
      pageSize: pagination.pageSize,
      pages,
      hasNext: pagination.page < pages,
      hasPrev: pagination.page > 1,
    };

    res.json(response);
  } catch (error) {
    if (error instanceof z.ZodError) {
      res.status(400).json({
        error: "ValidationError",
        message: "Invalid query parameters",
        details: error.errors,
      });
    } else {
      res.status(500).json({
        error: "InternalServerError",
        message: "An unexpected error occurred",
      });
    }
  }
});
```

### Pattern 3: Error Handling and Status Codes

```typescript
import { Request, Response, NextFunction } from "express";

// Error types
interface ErrorResponse {
  error: string;
  message: string;
  details?: Record<string, any>;
  timestamp: string;
  path: string;
}

interface ValidationErrorDetail {
  field: string;
  message: string;
  value?: any;
}

// Custom error classes
class ApiError extends Error {
  constructor(
    public statusCode: number,
    public error: string,
    message: string,
    public details?: Record<string, any>
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
}

class NotFoundError extends ApiError {
  constructor(resource: string, id: string) {
    super(404, "NotFound", `${resource} not found`, { id });
  }
}

class ValidationError extends ApiError {
  constructor(errors: ValidationErrorDetail[]) {
    super(422, "ValidationError", "Request validation failed", { errors });
  }
}

class UnauthorizedError extends ApiError {
  constructor(message: string = "Unauthorized") {
    super(401, "Unauthorized", message);
  }
}

class ForbiddenError extends ApiError {
  constructor(message: string = "Forbidden") {
    super(403, "Forbidden", message);
  }
}

class ConflictError extends ApiError {
  constructor(message: string, details?: Record<string, any>) {
    super(409, "Conflict", message, details);
  }
}

const HTTP_STATUS = {
  OK: 200,
  CREATED: 201,
  NO_CONTENT: 204,
  BAD_REQUEST: 400,
  UNAUTHORIZED: 401,
  FORBIDDEN: 403,
  NOT_FOUND: 404,
  CONFLICT: 409,
  UNPROCESSABLE_ENTITY: 422,
  INTERNAL_SERVER_ERROR: 500,
} as const;

// Error handling middleware
function errorHandler(
  err: Error,
  req: Request,
  res: Response,
  next: NextFunction
): void {
  if (err instanceof ApiError) {
    const errorResponse: ErrorResponse = {
      error: err.error,
      message: err.message,
      details: err.details,
      timestamp: new Date().toISOString(),
      path: req.path,
    };

    res.status(err.statusCode).json(errorResponse);
  } else {
    const errorResponse: ErrorResponse = {
      error: "InternalServerError",
      message: "An unexpected error occurred",
      timestamp: new Date().toISOString(),
      path: req.path,
    };

    res.status(HTTP_STATUS.INTERNAL_SERVER_ERROR).json(errorResponse);
  }
}

router.get(
  "/api/users/:id",
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const user = await fetchUser(req.params.id);

      if (!user) {
        throw new NotFoundError("User", req.params.id);
      }

      res.json(user);
    } catch (error) {
      next(error);
    }
  }
);

router.post(
  "/api/users",
  async (req: Request, res: Response, next: NextFunction) => {
    try {
      const validationResult = UserSchema.safeParse(req.body);

      if (!validationResult.success) {
        const errors = validationResult.error.errors.map((err) => ({
          field: err.path.join("."),
          message: err.message,
          value: err.path.reduce((obj, key) => obj?.[key], req.body),
        }));

        throw new ValidationError(errors);
      }

      const existing = await findUserByEmail(validationResult.data.email);
      if (existing) {
        throw new ConflictError("User with this email already exists", {
          email: validationResult.data.email,
        });
      }

      const user = await createUser(validationResult.data);
      res.status(HTTP_STATUS.CREATED).json(user);
    } catch (error) {
      next(error);
    }
  }
);

// Register error handler (must be last)
app.use(errorHandler);
```

## Best Practices

### REST APIs

1. **Consistent Naming**: Use plural nouns for collections (`/users`, not `/user`)
2. **Stateless**: Each request contains all necessary information
3. **Use HTTP Status Codes Correctly**: 2xx success, 4xx client errors, 5xx server errors
4. **Version Your API**: Plan for breaking changes from day one
5. **Pagination**: Always paginate large collections
6. **Rate Limiting**: Protect your API with rate limits
7. **Documentation**: Use OpenAPI/Swagger for interactive docs
8. **Input Validation**: Validate all inputs using libraries like Zod or class-validator
9. **Error Consistency**: Standardize error response format across all endpoints
10. **Security Headers**: Implement CORS, CSRF protection, and security headers

### TypeScript-Specific Best Practices

1. **Type Safety**: Use Zod or class-validator for runtime validation
2. **DTOs**: Create separate types for request/response objects
3. **Middleware Types**: Properly type Express middleware and request extensions
4. **Error Handling**: Use custom error classes with type guards
5. **Async/Await**: Prefer async/await over callbacks or raw promises

## Common Pitfalls

- **Over-fetching/Under-fetching (REST)**: Fixed in GraphQL but requires DataLoaders
- **Breaking Changes**: Version APIs or use deprecation strategies
- **Inconsistent Error Formats**: Standardize error responses
- **Missing Rate Limits**: APIs without limits are vulnerable to abuse
- **Poor Documentation**: Undocumented APIs frustrate developers
- **Ignoring HTTP Semantics**: POST for idempotent operations breaks expectations
- **Tight Coupling**: API structure shouldn't mirror database schema
- **Missing Validation**: Always validate and sanitize user inputs
- **Weak Type Safety**: Avoid `any` types, use strict TypeScript configuration

## Resources

- **references/rest-design.md** : Comprehensive REST API design guide
