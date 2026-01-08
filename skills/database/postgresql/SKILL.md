# PostgreSQL Table Design

## Core Principles

### 1. Primary Keys
Define a **PRIMARY KEY** for reference tables (users, orders, products, etc.). Not always needed for time-series/event/log data.

**When to use which type:**
- **`BIGINT GENERATED ALWAYS AS IDENTITY`** - Default choice for most tables
- **`UUID`** - Only when you need global uniqueness, opacity, or distributed systems

**Example:**
```sql
-- ✅ Good: Simple reference table
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email TEXT NOT NULL
);

-- ✅ Good: Distributed system needing globally unique IDs
CREATE TABLE events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  data JSONB NOT NULL
);

-- ✅ Good: Time-series data without PK
CREATE TABLE sensor_readings (
  device_id BIGINT NOT NULL,
  reading_time TIMESTAMPTZ NOT NULL,
  temperature DOUBLE PRECISION NOT NULL
);
```

### 2. Normalization
**Normalize first (to 3NF)** to eliminate data redundancy and update anomalies. Denormalize **only** for measured, high-ROI reads where join performance is proven problematic.

**Example:**
```sql
-- ❌ Bad: Denormalized, redundant data
CREATE TABLE orders (
  order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_email TEXT NOT NULL,
  user_name TEXT NOT NULL,
  user_address TEXT NOT NULL
);

-- ✅ Good: Normalized design
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email TEXT NOT NULL,
  name TEXT NOT NULL,
  address TEXT NOT NULL
);

CREATE TABLE orders (
  order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(user_id)
);
```

### 3. Null Safety and Defaults
Add **NOT NULL** everywhere it's semantically required; use **DEFAULT**s for common values.

**Example:**
```sql
CREATE TABLE posts (
  post_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title TEXT NOT NULL,  -- Always required
  content TEXT NOT NULL,  -- Always required
  status TEXT NOT NULL DEFAULT 'DRAFT',  -- Reasonable default
  published_at TIMESTAMPTZ,  -- NULL = not published yet
  view_count BIGINT NOT NULL DEFAULT 0  -- Start at zero
);
```

### 4. Index Strategic Columns
Create **indexes for access paths you actually query**:
- Primary keys and unique constraints (automatic)
- **Foreign key columns** (manual - PostgreSQL doesn't auto-create these!)
- Frequent filter conditions
- Sort keys
- Join columns

### 5. Choose Correct Data Types
- **TIMESTAMPTZ** for timestamps with timezone awareness
- **NUMERIC(p,s)** for money and exact decimal values
- **TEXT** for strings (not VARCHAR)
- **BIGINT** for integer values
- **DOUBLE PRECISION** for floating-point numbers

## PostgreSQL "Gotchas"

### Identifier Casing
Unquoted identifiers are automatically lowercased. Always use `snake_case` for table and column names.

```sql
-- ❌ Bad: Mixed case requires quotes everywhere
CREATE TABLE "UserProfiles" ("userId" BIGINT);
SELECT "userId" FROM "UserProfiles";  -- Must always quote

-- ✅ Good: Lowercase, no quotes needed
CREATE TABLE user_profiles (user_id BIGINT);
SELECT user_id FROM user_profiles;
```

### UNIQUE Constraints and NULLs
UNIQUE constraints allow multiple NULL values by default. Use `NULLS NOT DISTINCT` (PostgreSQL 15+) to allow only one NULL.

```sql
-- Default behavior: multiple NULLs allowed
CREATE TABLE contacts (
  email TEXT UNIQUE
);
INSERT INTO contacts VALUES (NULL);  -- OK
INSERT INTO contacts VALUES (NULL);  -- OK (multiple NULLs allowed)

-- Restrict to single NULL (PostgreSQL 15+)
CREATE TABLE contacts (
  email TEXT UNIQUE NULLS NOT DISTINCT
);
INSERT INTO contacts VALUES (NULL);  -- OK
INSERT INTO contacts VALUES (NULL);  -- ❌ ERROR: duplicate key
```

### Foreign Key Indexes
PostgreSQL **does not** automatically create indexes on foreign key columns. You must add them manually for optimal join performance.

```sql
CREATE TABLE orders (
  order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(user_id)
);

-- ✅ CRITICAL: Manually index the FK column
CREATE INDEX ON orders (user_id);
```

### No Silent Type Coercion
Length and precision overflows throw errors instead of silently truncating.

```sql
CREATE TABLE products (price NUMERIC(5,2));
INSERT INTO products VALUES (999.99);  -- ✅ OK
INSERT INTO products VALUES (9999.99);  -- ❌ ERROR: numeric field overflow
```

### ID Sequence Gaps Are Normal
Rollbacks, crashes, and concurrent transactions create gaps in ID sequences. This is expected PostgreSQL behavior—don't try to make IDs consecutive.

```sql
-- Normal sequence: 1, 2, 5, 6, 10, 11...
-- Gaps from rolled-back transactions, crashes, etc.
-- This is NORMAL and should not be "fixed"
```

### Heap Storage (No Clustered Index)
Unlike SQL Server or MySQL InnoDB, PostgreSQL has no clustered primary key by default. Row order on disk follows insertion order unless explicitly clustered.

```sql
-- One-time reorganization (not maintained on inserts)
CLUSTER users USING users_pkey;
```

### MVCC and Dead Tuples
Updates and deletes leave dead tuples that are cleaned up by VACUUM. Design to avoid frequently updating wide rows with many indexes.

## Data Types

### Identifiers (IDs)
**`BIGINT GENERATED ALWAYS AS IDENTITY`** - Preferred for most tables
- `GENERATED BY DEFAULT` also acceptable
- Auto-incrementing, efficient, simple

**`UUID`** - For distributed systems or when you need opacity
- Use `gen_random_uuid()` for PostgreSQL < 18
- Use `uuidv7()` for PostgreSQL 18+ (time-ordered UUIDs)

**Example:**
```sql
-- Most tables
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);

-- Distributed/federated systems
CREATE TABLE events (
  event_id UUID PRIMARY KEY DEFAULT gen_random_uuid()
);
```

### Numbers

**Integers:**
- **`BIGINT`** - Default choice (8 bytes, -9 quintillion to +9 quintillion)
- **`INTEGER`** - When storage is critical (4 bytes, -2 billion to +2 billion)
- Avoid `SMALLINT` unless severely constrained

**Decimals:**
- **`NUMERIC(precision, scale)`** - For money and exact arithmetic
- **`DOUBLE PRECISION`** - For scientific calculations and floats
- Never use `REAL` or floating-point types for money

**Example:**
```sql
CREATE TABLE products (
  product_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  price NUMERIC(10,2) NOT NULL,  -- Money: exact decimals
  weight_kg DOUBLE PRECISION,    -- Scientific: approximation OK
  quantity INTEGER NOT NULL       -- Count: smaller range OK
);
```

### Strings

**`TEXT`** - Preferred for all text columns (no length penalty)

**Avoid:** `VARCHAR(n)`, `CHAR(n)`

**If length limits needed:** Use `CHECK` constraint instead
```sql
CREATE TABLE users (
  username TEXT NOT NULL CHECK (LENGTH(username) BETWEEN 3 AND 30)
);
```

**Case-insensitive matching:**
```sql
-- Option 1: Expression index (preferred for most cases)
CREATE TABLE users (email TEXT NOT NULL);
CREATE UNIQUE INDEX ON users (LOWER(email));
SELECT * FROM users WHERE LOWER(email) = LOWER('User@Example.com');

-- Option 2: CITEXT extension (for case-insensitive constraints)
CREATE EXTENSION IF NOT EXISTS citext;
CREATE TABLE users (email CITEXT NOT NULL UNIQUE);
```

**Binary data:** Use `BYTEA`

### Timestamps

**`TIMESTAMPTZ`** - Always use this for timestamps (stores timezone info)
- `DATE` - Date only (no time component)
- `INTERVAL` - Durations

**Avoid:** `TIMESTAMP` (without timezone), `TIMETZ`

**Functions:**
- `now()` - Transaction start time (stable within transaction)
- `clock_timestamp()` - Current wall-clock time (changes during transaction)

**Example:**
```sql
CREATE TABLE events (
  event_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now(),
  event_date DATE NOT NULL,
  duration INTERVAL
);

INSERT INTO events (event_date, duration)
VALUES ('2025-01-15', INTERVAL '2 hours 30 minutes');
```

### Booleans
**`BOOLEAN`** with `NOT NULL` unless you truly need three states (true/false/unknown)

```sql
CREATE TABLE users (
  is_active BOOLEAN NOT NULL DEFAULT true,
  email_verified BOOLEAN  -- NULL = unknown, true = verified, false = bounced
);
```

### Enums
**When to use:**
- Small, stable sets (US states, days of week, currencies)
- Values rarely change

**When NOT to use:**
- Business logic that evolves (order statuses, user roles)
- Use `TEXT` + `CHECK` or lookup table instead

**Example:**
```sql
-- ✅ Good: Stable domain
CREATE TYPE day_of_week AS ENUM ('Mon', 'Tue', 'Wed', 'Thu', 'Fri', 'Sat', 'Sun');

-- ❌ Bad: Business logic (use TEXT + CHECK instead)
CREATE TYPE order_status AS ENUM ('PENDING', 'PAID', 'SHIPPED');  -- Will change!

-- ✅ Better: Flexible for business changes
CREATE TABLE orders (
  status TEXT NOT NULL DEFAULT 'PENDING'
    CHECK (status IN ('PENDING', 'PAID', 'SHIPPED', 'DELIVERED', 'CANCELLED'))
);
```

### Arrays
**`TEXT[]`, `INTEGER[]`, `JSONB[]`** - For ordered lists

**Index with GIN** for containment and overlap queries

**Example:**
```sql
CREATE TABLE articles (
  article_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  tags TEXT[] NOT NULL DEFAULT '{}'
);

CREATE INDEX ON articles USING GIN (tags);

-- Queries
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];  -- Contains
SELECT * FROM articles WHERE tags && ARRAY['sql', 'database'];  -- Overlaps
SELECT * FROM articles WHERE 'postgresql' = ANY(tags);  -- Element match

-- Array access (1-indexed!)
SELECT tags[1], tags[2:4] FROM articles;
```

### Range Types
**`daterange`, `tstzrange`, `int4range`, `numrange`** - For intervals

**Index with GiST**

**Example:**
```sql
CREATE TABLE room_bookings (
  booking_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id BIGINT NOT NULL,
  booking_period TSTZRANGE NOT NULL,
  EXCLUDE USING gist (room_id WITH =, booking_period WITH &&)  -- Prevent overlaps
);

INSERT INTO room_bookings (room_id, booking_period)
VALUES (101, '[2025-01-15 14:00, 2025-01-15 16:00)');

-- Queries
SELECT * FROM room_bookings
WHERE booking_period @> '2025-01-15 15:00'::TIMESTAMPTZ;  -- Contains timestamp

SELECT * FROM room_bookings
WHERE booking_period && '[2025-01-15 13:00, 2025-01-15 17:00)'::TSTZRANGE;  -- Overlaps
```

### Network Types
**`INET`** - IP addresses (v4 or v6)
**`CIDR`** - Network ranges
**`MACADDR`** - MAC addresses

**Example:**
```sql
CREATE TABLE servers (
  server_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  ip_address INET NOT NULL,
  network CIDR NOT NULL,
  mac_address MACADDR
);

-- Queries
SELECT * FROM servers WHERE ip_address << '192.168.1.0/24'::CIDR;  -- IP in network
```

### Full-Text Search
**`TSVECTOR`** - Indexed document
**`TSQUERY`** - Search query

**Always specify language** (never use single-argument versions)

**Example:**
```sql
CREATE TABLE articles (
  article_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  title TEXT NOT NULL,
  content TEXT NOT NULL,
  search_vector TSVECTOR GENERATED ALWAYS AS (
    to_tsvector('english', title || ' ' || content)
  ) STORED
);

CREATE INDEX ON articles USING GIN (search_vector);

SELECT * FROM articles
WHERE search_vector @@ to_tsquery('english', 'postgresql & performance');
```

### Custom Types

**Domain types** - Reusable types with validation
```sql
CREATE DOMAIN email AS TEXT
  CHECK (VALUE ~ '^[^@]+@[^@]+\.[^@]+$');

CREATE DOMAIN us_postal_code AS TEXT
  CHECK (VALUE ~ '^\d{5}(-\d{4})?$');

CREATE TABLE contacts (
  email email NOT NULL,
  zip_code us_postal_code
);
```

**Composite types** - Structured data within columns
```sql
CREATE TYPE address AS (
  street TEXT,
  city TEXT,
  state TEXT,
  zip TEXT
);

CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  shipping_address address
);

SELECT (shipping_address).city FROM users;
```

### JSONB
**Preferred over JSON** for all use cases except when key order must be preserved.

**Index with GIN**

**Use for:** Optional/semi-structured attributes, configuration

**Example:**
```sql
CREATE TABLE user_profiles (
  user_id BIGINT PRIMARY KEY,
  preferences JSONB NOT NULL DEFAULT '{}',
  metadata JSONB
);

CREATE INDEX ON user_profiles USING GIN (preferences);

-- Queries
SELECT * FROM user_profiles
WHERE preferences @> '{"theme": "dark"}';  -- Containment

SELECT * FROM user_profiles
WHERE preferences ? 'notifications';  -- Key exists
```

### Specialized Types

**`vector`** (pgvector extension) - For embeddings and similarity search
```sql
CREATE EXTENSION vector;

CREATE TABLE documents (
  doc_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  embedding vector(1536)  -- OpenAI ada-002 dimension
);

CREATE INDEX ON documents USING ivfflat (embedding vector_cosine_ops);
```

### Do not use the following data types

- DO NOT use `timestamp` (without time zone); DO use `timestamptz` instead.
- DO NOT use `char(n)` or `varchar(n)`; DO use `text` instead.
- DO NOT use `money` type; DO use `numeric` instead.
- DO NOT use `timetz` type; DO use `timestamptz` instead.
- DO NOT use `timestamptz(0)` or any other precision specification; DO use `timestamptz` instead
- DO NOT use `serial` type; DO use `generated always as identity` instead.

## Table Types

- **Regular**: default; fully durable, logged.
- **TEMPORARY**: session-scoped, auto-dropped, not logged. Faster for scratch work.
- **UNLOGGED**: persistent but not crash-safe. Faster writes; good for caches/staging.

## Row-Level Security

Enable with `ALTER TABLE tbl ENABLE ROW LEVEL SECURITY`. Create policies: `CREATE POLICY user_access ON orders FOR SELECT TO app_users USING (user_id = current_user_id())`. Built-in user-based access control at the row level.

## Constraints

### PRIMARY KEY
Enforces `UNIQUE` + `NOT NULL` and automatically creates a B-tree index.

```sql
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

### FOREIGN KEY
Always specify `ON DELETE` and `ON UPDATE` actions. **Manually index FK columns** for performance.

**Actions:**
- `CASCADE` - Delete/update child rows when parent changes
- `RESTRICT` - Prevent parent changes if children exist (default)
- `SET NULL` - Set child FK to NULL when parent deleted
- `SET DEFAULT` - Set child FK to default value

**Example:**
```sql
CREATE TABLE orders (
  order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(user_id) ON DELETE CASCADE,
  product_id BIGINT NOT NULL REFERENCES products(product_id) ON DELETE RESTRICT
);

-- ✅ CRITICAL: Index FK columns manually
CREATE INDEX ON orders (user_id);
CREATE INDEX ON orders (product_id);
```

**Circular dependencies:**
```sql
CREATE TABLE employees (
  emp_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  manager_id BIGINT REFERENCES employees(emp_id) DEFERRABLE INITIALLY DEFERRED
);

-- This transaction works because constraint checked at commit
BEGIN;
INSERT INTO employees (emp_id, manager_id) OVERRIDING SYSTEM VALUE VALUES (1, 2);
INSERT INTO employees (emp_id, manager_id) OVERRIDING SYSTEM VALUE VALUES (2, 1);
COMMIT;
```

### UNIQUE
Creates a B-tree index. Allows multiple NULL values unless `NULLS NOT DISTINCT` (PostgreSQL 15+).

```sql
-- Standard: Multiple NULLs allowed
CREATE TABLE products (
  sku TEXT UNIQUE,  -- NULL, NULL, NULL all allowed
  name TEXT NOT NULL
);

-- PostgreSQL 15+: Only one NULL allowed
CREATE TABLE products (
  sku TEXT UNIQUE NULLS NOT DISTINCT,
  name TEXT NOT NULL
);
```

### CHECK
Row-level validation. **Important:** NULL values always pass CHECK constraints (SQL three-valued logic).

```sql
-- ❌ This allows NULL prices!
CREATE TABLE products (
  price NUMERIC CHECK (price > 0)
);

-- ✅ Combine with NOT NULL to enforce
CREATE TABLE products (
  price NUMERIC NOT NULL CHECK (price > 0),
  discount_pct NUMERIC CHECK (discount_pct BETWEEN 0 AND 100)
);
```

### EXCLUDE
Prevents overlapping values using custom operators. Commonly used with ranges to prevent conflicts.

**Example:**
```sql
CREATE TABLE room_bookings (
  booking_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  room_id BIGINT NOT NULL,
  booking_period TSTZRANGE NOT NULL,
  -- Prevent double-booking the same room
  EXCLUDE USING gist (room_id WITH =, booking_period WITH &&)
);

-- This insert succeeds
INSERT INTO room_bookings (room_id, booking_period)
VALUES (101, '[2025-01-15 14:00, 2025-01-15 16:00)');

-- This fails: overlaps with previous booking
INSERT INTO room_bookings (room_id, booking_period)
VALUES (101, '[2025-01-15 15:00, 2025-01-15 17:00)');  -- ❌ ERROR
```

## Indexing

### B-tree (Default)
For equality, range queries, sorting: `=`, `<`, `>`, `BETWEEN`, `ORDER BY`

```sql
CREATE INDEX ON users (email);
CREATE INDEX ON orders (created_at);
```

### Composite Indexes
**Column order matters!** Index is used only when filtering on leftmost prefix.

```sql
CREATE INDEX ON orders (user_id, created_at);

-- ✅ Uses index: filters on leftmost column
SELECT * FROM orders WHERE user_id = 123;
SELECT * FROM orders WHERE user_id = 123 AND created_at > '2025-01-01';

-- ❌ Does NOT use index: skips leftmost column
SELECT * FROM orders WHERE created_at > '2025-01-01';
```

**Rule of thumb:** Put most selective/frequently filtered columns first.

### Covering Indexes
Include non-key columns to avoid table lookups (index-only scans).

```sql
CREATE INDEX ON users (email) INCLUDE (name, created_at);

-- Index-only scan: all columns in index, no table access needed
SELECT name, created_at FROM users WHERE email = 'user@example.com';
```

### Partial Indexes
Index only a subset of rows. Smaller, faster, cheaper to maintain.

```sql
-- Only index active users
CREATE INDEX ON users (email) WHERE status = 'active';

-- This query uses the partial index
SELECT * FROM users WHERE email = 'user@example.com' AND status = 'active';
```

**Common use cases:**
- `WHERE deleted_at IS NULL` (soft deletes)
- `WHERE status = 'active'`
- `WHERE published = true`

### Expression Indexes
Index computed values. Expression in query must match exactly.

```sql
CREATE INDEX ON users (LOWER(email));

-- ✅ Uses index
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- ❌ Does NOT use index
SELECT * FROM users WHERE email = 'user@example.com';
```

### GIN (Generalized Inverted Index)
For JSONB, arrays, full-text search.

```sql
-- JSONB
CREATE INDEX ON user_profiles USING GIN (preferences);
SELECT * FROM user_profiles WHERE preferences @> '{"theme": "dark"}';

-- Arrays
CREATE INDEX ON articles USING GIN (tags);
SELECT * FROM articles WHERE tags @> ARRAY['postgresql'];

-- Full-text search
CREATE INDEX ON articles USING GIN (search_vector);
SELECT * FROM articles WHERE search_vector @@ to_tsquery('english', 'database');
```

### GiST (Generalized Search Tree)
For ranges, geometry, exclusion constraints.

```sql
CREATE INDEX ON bookings USING gist (booking_period);
SELECT * FROM bookings WHERE booking_period && '[2025-01-15, 2025-01-20)'::TSTZRANGE;
```

### BRIN (Block Range Index)
For very large, naturally ordered tables (time-series). Minimal storage overhead.

```sql
-- Effective for time-series data inserted in chronological order
CREATE INDEX ON sensor_readings USING brin (timestamp);

-- Much smaller than B-tree, fast enough for time-range queries
SELECT * FROM sensor_readings WHERE timestamp BETWEEN '2025-01-01' AND '2025-01-31';
```

**Use when:**
- Table is very large (100M+ rows)
- Data is naturally ordered (timestamps, sequential IDs)
- You query by range on ordered column

## Partitioning

### When to Partition
- Very large tables (100M+ rows)
- Queries consistently filter on partition key (usually time/date)
- Data maintenance needs (periodic pruning, bulk replacement)

### Range Partitioning
Common for time-series data.

```sql
CREATE TABLE logs (
  log_id BIGINT GENERATED ALWAYS AS IDENTITY,
  created_at TIMESTAMPTZ NOT NULL,
  message TEXT NOT NULL
) PARTITION BY RANGE (created_at);

-- Create monthly partitions
CREATE TABLE logs_2025_01 PARTITION OF logs
  FOR VALUES FROM ('2025-01-01') TO ('2025-02-01');

CREATE TABLE logs_2025_02 PARTITION OF logs
  FOR VALUES FROM ('2025-02-01') TO ('2025-03-01');

-- Query automatically uses only relevant partition
SELECT * FROM logs WHERE created_at >= '2025-01-15';  -- Only scans logs_2025_01
```

**Consider TimescaleDB** for automatic time-based partitioning with retention policies and compression.

### List Partitioning
For discrete, categorical values.

```sql
CREATE TABLE sales (
  sale_id BIGINT GENERATED ALWAYS AS IDENTITY,
  region TEXT NOT NULL,
  amount NUMERIC NOT NULL
) PARTITION BY LIST (region);

CREATE TABLE sales_us_east PARTITION OF sales
  FOR VALUES IN ('us-east-1', 'us-east-2');

CREATE TABLE sales_us_west PARTITION OF sales
  FOR VALUES IN ('us-west-1', 'us-west-2');

CREATE TABLE sales_eu PARTITION OF sales
  FOR VALUES IN ('eu-west-1', 'eu-central-1');
```

### Hash Partitioning
For even distribution when no natural partitioning key.

```sql
CREATE TABLE user_events (
  event_id BIGINT GENERATED ALWAYS AS IDENTITY,
  user_id BIGINT NOT NULL,
  event_data JSONB NOT NULL
) PARTITION BY HASH (user_id);

-- Create 4 partitions (0, 1, 2, 3)
CREATE TABLE user_events_0 PARTITION OF user_events
  FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE user_events_1 PARTITION OF user_events
  FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE user_events_2 PARTITION OF user_events
  FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE user_events_3 PARTITION OF user_events
  FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

### Important Limitations

**No global UNIQUE constraints** - Must include partition key in unique constraint:
```sql
-- ❌ This fails
CREATE TABLE logs (
  log_id BIGINT PRIMARY KEY,
  created_at TIMESTAMPTZ NOT NULL
) PARTITION BY RANGE (created_at);

-- ✅ This works
CREATE TABLE logs (
  log_id BIGINT,
  created_at TIMESTAMPTZ NOT NULL,
  PRIMARY KEY (log_id, created_at)  -- Must include partition key
) PARTITION BY RANGE (created_at);
```

**Avoid table inheritance** - Use declarative partitioning (PostgreSQL 10+) or TimescaleDB hypertables.

## Special Considerations

### Update-Heavy Tables

- **Separate hot/cold columns**—put frequently updated columns in separate table to minimize bloat.
- **Use `fillfactor=90`** to leave space for HOT updates that avoid index maintenance.
- **Avoid updating indexed columns**—prevents beneficial HOT updates.
- **Partition by update patterns**—separate frequently updated rows in a different partition from stable data.

### Insert-Heavy Workloads

- **Minimize indexes**—only create what you query; every index slows inserts.
- **Use `COPY` or multi-row `INSERT`** instead of single-row inserts.
- **UNLOGGED tables** for rebuildable staging data—much faster writes.
- **Defer index creation** for bulk loads—>drop index, load data, recreate indexes.
- **Partition by time/hash** to distribute load. **TimescaleDB** automates partitioning and compression of insert-heavy data.
- **Use a natural key for primary key** such as a (timestamp, device_id) if enforcing global uniqueness is important many insert-heavy tables don't need a primary key at all.
- If you do need a surrogate key, **Prefer `BIGINT GENERATED ALWAYS AS IDENTITY` over `UUID`**.

### Upsert-Friendly Design

- **Requires UNIQUE index** on conflict target columns—`ON CONFLICT (col1, col2)` needs exact matching unique index (partial indexes don't work).
- **Use `EXCLUDED.column`** to reference would-be-inserted values; only update columns that actually changed to reduce write overhead.
- **`DO NOTHING` faster** than `DO UPDATE` when no actual update needed.

### Safe Schema Evolution

- **Transactional DDL**: most DDL operations can run in transactions and be rolled back—`BEGIN; ALTER TABLE...; ROLLBACK;` for safe testing.
- **Concurrent index creation**: `CREATE INDEX CONCURRENTLY` avoids blocking writes but can't run in transactions.
- **Volatile defaults cause rewrites**: adding `NOT NULL` columns with volatile defaults (e.g., `now()`, `gen_random_uuid()`) rewrites entire table. Non-volatile defaults are fast.
- **Drop constraints before columns**: `ALTER TABLE DROP CONSTRAINT` then `DROP COLUMN` to avoid dependency issues.
- **Function signature changes**: `CREATE OR REPLACE` with different arguments creates overloads, not replacements. DROP old version if no overload desired.

## Generated Columns

- `... GENERATED ALWAYS AS (<expr>) STORED` for computed, indexable fields. PG18+ adds `VIRTUAL` columns (computed on read, not stored).

## Extensions

- **`pgcrypto`**: `crypt()` for password hashing.
- **`uuid-ossp`**: alternative UUID functions; prefer `pgcrypto` for new projects.
- **`pg_trgm`**: fuzzy text search with `%` operator, `similarity()` function. Index with GIN for `LIKE '%pattern%'` acceleration.
- **`citext`**: case-insensitive text type. Prefer expression indexes on `LOWER(col)` unless you need case-insensitive constraints.
- **`btree_gin`/`btree_gist`**: enable mixed-type indexes (e.g., GIN index on both JSONB and text columns).
- **`hstore`**: key-value pairs; mostly superseded by JSONB but useful for simple string mappings.
- **`timescaledb`**: essential for time-series—automated partitioning, retention, compression, continuous aggregates.
- **`postgis`**: comprehensive geospatial support beyond basic geometric types—essential for location-based applications.
- **`pgvector`**: vector similarity search for embeddings.
- **`pgaudit`**: audit logging for all database activity.

## JSONB Guidance

- Prefer `JSONB` with **GIN** index.
- Default: `CREATE INDEX ON tbl USING GIN (jsonb_col);` → accelerates:
  - **Containment** `jsonb_col @> '{"k":"v"}'`
  - **Key existence** `jsonb_col ? 'k'`, **any/all keys** `?\|`, `?&`
  - **Path containment** on nested docs
  - **Disjunction** `jsonb_col @> ANY(ARRAY['{"status":"active"}', '{"status":"pending"}'])`
- Heavy `@>` workloads: consider opclass `jsonb_path_ops` for smaller/faster containment-only indexes:
  - `CREATE INDEX ON tbl USING GIN (jsonb_col jsonb_path_ops);`
  - **Trade-off**: loses support for key existence (`?`, `?|`, `?&`) queries—only supports containment (`@>`)
- Equality/range on a specific scalar field: extract and index with B-tree (generated column or expression):
  - `ALTER TABLE tbl ADD COLUMN price INT GENERATED ALWAYS AS ((jsonb_col->>'price')::INT) STORED;`
  - `CREATE INDEX ON tbl (price);`
  - Prefer queries like `WHERE price BETWEEN 100 AND 500` (uses B-tree) over `WHERE (jsonb_col->>'price')::INT BETWEEN 100 AND 500` without index.
- Arrays inside JSONB: use GIN + `@>` for containment (e.g., tags). Consider `jsonb_path_ops` if only doing containment.
- Keep core relations in tables; use JSONB for optional/variable attributes.
- Use constraints to limit allowed JSONB values in a column e.g. `config JSONB NOT NULL CHECK(jsonb_typeof(config) = 'object')`

## Examples

### Users

```sql
CREATE TABLE users (
  user_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  email TEXT NOT NULL UNIQUE,
  name TEXT NOT NULL,
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE UNIQUE INDEX ON users (LOWER(email));
CREATE INDEX ON users (created_at);
```

### Orders

```sql
CREATE TABLE orders (
  order_id BIGINT GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  user_id BIGINT NOT NULL REFERENCES users(user_id),
  status TEXT NOT NULL DEFAULT 'PENDING' CHECK (status IN ('PENDING','PAID','CANCELED')),
  total NUMERIC(10,2) NOT NULL CHECK (total > 0),
  created_at TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX ON orders (user_id);
CREATE INDEX ON orders (created_at);
```

### JSONB

```sql
CREATE TABLE profiles (
  user_id BIGINT PRIMARY KEY REFERENCES users(user_id),
  attrs JSONB NOT NULL DEFAULT '{}',
  theme TEXT GENERATED ALWAYS AS (attrs->>'theme') STORED
);
CREATE INDEX profiles_attrs_gin ON profiles USING GIN (attrs);
```
