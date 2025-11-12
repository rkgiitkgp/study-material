# PostgreSQL & Data Layer - Interview Answers

## Question 1: Explain indexing strategies — B-Tree, GIN, and GiST — and when to use them.

### Conceptual Explanation

PostgreSQL offers multiple index types for different use cases. **B-Tree** is the default and works for equality and range queries on scalar values (numbers, strings, dates). It's ordered, making it perfect for `<`, `>`, `BETWEEN`, `ORDER BY` operations.

**GIN (Generalized Inverted Index)** is designed for indexing composite values like arrays, JSONB, and full-text search. It stores a separate entry for each element/key within the value, making it efficient for "contains" queries (`@>`, `?`, text search).

**GiST (Generalized Search Tree)** is flexible and supports geometric data, network addresses, full-text search, and custom types. It's used for spatial queries (PostGIS), nearest-neighbor searches, and range types. It's generally larger and slower than GIN but more versatile.

### Code Example

```sql
-- B-Tree indexes (default)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created_at ON orders(created_at);
CREATE INDEX idx_users_last_name_first_name ON users(last_name, first_name); -- Composite

-- Queries that use B-Tree
SELECT * FROM users WHERE email = 'user@example.com'; -- Exact match
SELECT * FROM orders WHERE created_at > '2024-01-01'; -- Range
SELECT * FROM users ORDER BY last_name, first_name LIMIT 10; -- Ordered retrieval

-- GIN indexes for JSONB and arrays
CREATE INDEX idx_products_tags ON products USING GIN(tags); -- Array column
CREATE INDEX idx_users_metadata ON users USING GIN(metadata); -- JSONB column
CREATE INDEX idx_posts_search ON posts USING GIN(to_tsvector('english', content)); -- Full-text

-- Queries that use GIN
SELECT * FROM products WHERE tags @> ARRAY['electronics', 'sale']; -- Array contains
SELECT * FROM users WHERE metadata @> '{"premium": true}'::jsonb; -- JSONB contains
SELECT * FROM users WHERE metadata ? 'verified'; -- JSONB key exists
SELECT * FROM posts WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & performance');

-- GiST indexes for geometric and range types
CREATE INDEX idx_locations_point ON locations USING GIST(coordinates); -- PostGIS
CREATE INDEX idx_events_period ON events USING GIST(time_range); -- Range type

-- Queries that use GiST
SELECT * FROM locations WHERE coordinates <-> point(40.7128, -74.0060) < 10; -- Distance
SELECT * FROM events WHERE time_range && daterange('2024-01-01', '2024-01-31'); -- Overlap

-- Partial indexes (for frequently filtered subsets)
CREATE INDEX idx_active_users ON users(last_login) WHERE active = true;
CREATE INDEX idx_pending_orders ON orders(created_at) WHERE status = 'pending';

-- Expression indexes
CREATE INDEX idx_users_lower_email ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

-- EXPLAIN to verify index usage
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'test@example.com';
```

### Best Practices

- **Use B-Tree for most cases**: It's the default for good reason—handles equality, ranges, sorting efficiently. Start here unless you have specific needs.
- **GIN for containment queries**: Perfect for JSONB, arrays, and full-text search. Beware: GIN indexes can be large and slower to update.
- **Partial indexes save space**: If you frequently query a subset (e.g., active users, pending orders), partial indexes are smaller and faster.
- **Composite index order matters**: For `idx(a, b)`, queries can use it for `WHERE a = ?` or `WHERE a = ? AND b = ?`, but not for `WHERE b = ?` alone.
- **Monitor index usage**: Use `pg_stat_user_indexes` to find unused indexes. They slow down writes without providing value.

---

## Question 2: How would you handle large tables with millions of rows (partitioning, indexing)?

### Conceptual Explanation

Large tables (>10M rows) cause slow queries, large index sizes, and difficult maintenance operations. Solutions include **partitioning** (splitting table into smaller physical tables), **indexing strategy** (covering indexes, partial indexes), **archiving old data**, and **table maintenance** (VACUUM, ANALYZE).

**Partitioning** divides a table into smaller chunks based on a key (date ranges, hash of ID). Queries that filter on the partition key only scan relevant partitions. Common strategies: time-based (by month/year), hash-based (distribute evenly), or range-based (by ID ranges).

Before partitioning, optimize: ensure proper indexes exist, use EXPLAIN ANALYZE to identify bottlenecks, tune PostgreSQL parameters (work_mem, shared_buffers), and consider if archiving old data solves the problem more simply.

### Code Example

```sql
-- Time-based partitioning (most common)
CREATE TABLE orders (
  id UUID DEFAULT gen_random_uuid(),
  user_id UUID NOT NULL,
  total DECIMAL(10,2),
  created_at TIMESTAMP WITH TIME ZONE NOT NULL,
  status VARCHAR(20)
) PARTITION BY RANGE (created_at);

-- Create partitions
CREATE TABLE orders_2024_01 PARTITION OF orders
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
  FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
  FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Indexes on each partition
CREATE INDEX idx_orders_2024_01_user ON orders_2024_01(user_id);
CREATE INDEX idx_orders_2024_02_user ON orders_2024_02(user_id);

-- Queries automatically use correct partition
SELECT * FROM orders WHERE created_at >= '2024-02-01' AND created_at < '2024-02-15';
-- Only scans orders_2024_02 partition

-- Hash partitioning for even distribution
CREATE TABLE users (
  id UUID DEFAULT gen_random_uuid(),
  email VARCHAR(255) NOT NULL,
  name VARCHAR(255),
  created_at TIMESTAMP WITH TIME ZONE
) PARTITION BY HASH (id);

CREATE TABLE users_p0 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE users_p1 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE users_p2 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE users_p3 PARTITION OF users FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Automatic partition management (pg_partman extension)
CREATE EXTENSION pg_partman;

SELECT create_parent(
  'public.orders',
  'created_at',
  'native',
  'monthly',
  p_start_partition := '2024-01-01'
);

-- Archiving old data
CREATE TABLE orders_archive (LIKE orders INCLUDING ALL);

-- Move old data
INSERT INTO orders_archive 
SELECT * FROM orders WHERE created_at < '2023-01-01';

DELETE FROM orders WHERE created_at < '2023-01-01';

-- Covering indexes (include columns for index-only scans)
CREATE INDEX idx_orders_user_covering 
  ON orders(user_id) INCLUDE (total, status, created_at);

-- Query can be satisfied entirely from index
SELECT total, status, created_at 
FROM orders 
WHERE user_id = 'some-uuid';
```

### Best Practices

- **Partition by time for time-series data**: Logs, orders, events benefit from monthly/yearly partitions. Makes archiving and deletion fast (drop partition vs DELETE).
- **Keep partitions manageable**: Aim for 10-100GB per partition. Too many small partitions increase planning overhead.
- **Automate partition creation**: Use pg_partman or scheduled jobs to create future partitions. Prevents errors when time advances.
- **Index partition keys**: Even though queries filter on partition key, index it for performance within each partition.
- **Regular VACUUM and ANALYZE**: Especially after bulk operations. Keeps statistics accurate and reclaims space.
- **Consider columnar storage**: For analytics queries on very large tables, consider Citus columnar or migrate to TimescaleDB.

---

## Question 3: How do transactions and isolation levels work in Postgres?

### Conceptual Explanation

Transactions ensure ACID properties—a group of operations either all succeed or all fail. PostgreSQL uses **MVCC (Multi-Version Concurrency Control)** which allows readers not to block writers and vice versa by maintaining multiple versions of rows.

PostgreSQL supports four isolation levels: **Read Uncommitted** (behaves like Read Committed in Postgres), **Read Committed** (default—sees committed changes from other transactions), **Repeatable Read** (snapshot isolation—sees consistent data throughout transaction), and **Serializable** (strictest—prevents all anomalies, may cause serialization errors).

Choose isolation level based on consistency needs. Read Committed is fine for most OLTP. Repeatable Read prevents phantom reads and is good for reports. Serializable ensures true isolation but can cause more transaction conflicts requiring retries.

### Code Example

```sql
-- Basic transaction
BEGIN;

UPDATE accounts SET balance = balance - 100 WHERE user_id = 'user1';
UPDATE accounts SET balance = balance + 100 WHERE user_id = 'user2';

COMMIT; -- Or ROLLBACK to undo

-- Read Committed (default)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

BEGIN;
SELECT balance FROM accounts WHERE user_id = 'user1'; -- Sees committed changes from other transactions
-- If another transaction commits here, next SELECT sees the new value
SELECT balance FROM accounts WHERE user_id = 'user1'; -- Might be different
COMMIT;

-- Repeatable Read (snapshot isolation)
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;

BEGIN;
SELECT balance FROM accounts WHERE user_id = 'user1'; -- Takes snapshot
-- Even if another transaction commits changes
SELECT balance FROM accounts WHERE user_id = 'user1'; -- Same value as first SELECT
COMMIT;

-- Serializable (strictest)
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

BEGIN;
SELECT * FROM products WHERE quantity > 0;
UPDATE products SET quantity = quantity - 1 WHERE id = 'prod1';
COMMIT; -- May fail with serialization error if conflict detected

-- Handling serialization failures
const maxRetries = 3;
for (let attempt = 0; attempt < maxRetries; attempt++) {
  try {
    await db.query('BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE');
    
    const result = await db.query('SELECT quantity FROM products WHERE id = $1', [productId]);
    if (result.rows[0].quantity < 1) throw new Error('Out of stock');
    
    await db.query('UPDATE products SET quantity = quantity - 1 WHERE id = $1', [productId]);
    await db.query('INSERT INTO orders (product_id, user_id) VALUES ($1, $2)', [productId, userId]);
    
    await db.query('COMMIT');
    break; // Success
    
  } catch (error) {
    await db.query('ROLLBACK');
    
    if (error.code === '40001' && attempt < maxRetries - 1) {
      // Serialization failure - retry
      await sleep(Math.pow(2, attempt) * 100);
      continue;
    }
    throw error;
  }
}

-- Locking strategies
BEGIN;

-- FOR UPDATE: Lock rows for update (exclusive lock)
SELECT * FROM accounts WHERE user_id = 'user1' FOR UPDATE;
UPDATE accounts SET balance = balance - 100 WHERE user_id = 'user1';

COMMIT;

-- FOR SHARE: Lock rows to prevent modification but allow reads
SELECT * FROM orders WHERE id = 'order1' FOR SHARE;

-- SKIP LOCKED: Skip locked rows (useful for job queues)
SELECT * FROM jobs WHERE status = 'pending' LIMIT 10 FOR UPDATE SKIP LOCKED;
```

### Best Practices

- **Use Read Committed for most operations**: It's the default and provides good balance between consistency and concurrency.
- **Repeatable Read for reports**: Ensures consistent view of data throughout long-running analytical queries.
- **Keep transactions short**: Long transactions hold locks and old MVCC versions, causing bloat and blocking others.
- **Handle serialization failures**: With Serializable isolation, implement retry logic for `40001` (serialization_failure) errors.
- **Use FOR UPDATE carefully**: It creates exclusive locks. Consider FOR UPDATE SKIP LOCKED for job queue patterns.
- **Avoid explicit LOCK TABLE**: Rarely needed and causes significant blocking. Use row-level locks instead.

---

## Question 4: How would you design schema for time-series or audit logs?

### Conceptual Explanation

Time-series data (metrics, logs, events) has unique characteristics: high write volume, time-based queries, old data rarely updated, and data grows continuously. Design priorities: efficient writes, fast time-range queries, and easy data retention management.

Key strategies: **time-based partitioning** (drop old partitions instead of DELETE), **append-only design** (no updates/deletes), **minimal indexes** (only on time + frequently filtered columns), and **data retention policies**. For very high volumes, consider TimescaleDB extension or separate OLAP database.

Audit logs track who changed what and when. Requirements: immutability, efficient queries by entity and user, and compliance with retention policies. Use JSONB for flexible schema and GIN indexes for querying.

### Code Example

```sql
-- Time-series metrics table with partitioning
CREATE TABLE metrics (
  time TIMESTAMP WITH TIME ZONE NOT NULL,
  metric_name VARCHAR(100) NOT NULL,
  value DOUBLE PRECISION NOT NULL,
  tags JSONB,
  host VARCHAR(100)
) PARTITION BY RANGE (time);

-- Create monthly partitions
CREATE TABLE metrics_2024_01 PARTITION OF metrics
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Indexes for fast queries
CREATE INDEX idx_metrics_2024_01_time ON metrics_2024_01(time DESC);
CREATE INDEX idx_metrics_2024_01_name_time ON metrics_2024_01(metric_name, time DESC);
CREATE INDEX idx_metrics_2024_01_tags ON metrics_2024_01 USING GIN(tags);

-- Efficient time-range queries
SELECT metric_name, AVG(value) as avg_value
FROM metrics
WHERE time >= NOW() - INTERVAL '1 hour'
  AND metric_name = 'cpu_usage'
GROUP BY metric_name;

-- Drop old partitions (instant, vs slow DELETE)
DROP TABLE metrics_2023_01;

-- Audit log table
CREATE TABLE audit_logs (
  id BIGSERIAL PRIMARY KEY,
  timestamp TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
  user_id UUID NOT NULL,
  action VARCHAR(50) NOT NULL,
  entity_type VARCHAR(50) NOT NULL,
  entity_id UUID NOT NULL,
  old_values JSONB,
  new_values JSONB,
  ip_address INET,
  user_agent TEXT
) PARTITION BY RANGE (timestamp);

CREATE TABLE audit_logs_2024_01 PARTITION OF audit_logs
  FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Indexes for common queries
CREATE INDEX idx_audit_logs_2024_01_timestamp ON audit_logs_2024_01(timestamp DESC);
CREATE INDEX idx_audit_logs_2024_01_user ON audit_logs_2024_01(user_id, timestamp DESC);
CREATE INDEX idx_audit_logs_2024_01_entity ON audit_logs_2024_01(entity_type, entity_id, timestamp DESC);

-- Trigger for automatic audit logging
CREATE OR REPLACE FUNCTION audit_changes()
RETURNS TRIGGER AS $$
BEGIN
  IF TG_OP = 'UPDATE' THEN
    INSERT INTO audit_logs (user_id, action, entity_type, entity_id, old_values, new_values)
    VALUES (
      current_setting('app.user_id')::uuid,
      'UPDATE',
      TG_TABLE_NAME,
      OLD.id,
      to_jsonb(OLD),
      to_jsonb(NEW)
    );
  ELSIF TG_OP = 'DELETE' THEN
    INSERT INTO audit_logs (user_id, action, entity_type, entity_id, old_values)
    VALUES (
      current_setting('app.user_id')::uuid,
      'DELETE',
      TG_TABLE_NAME,
      OLD.id,
      to_jsonb(OLD)
    );
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Apply trigger to tables
CREATE TRIGGER audit_users
AFTER UPDATE OR DELETE ON users
FOR EACH ROW EXECUTE FUNCTION audit_changes();

-- Application sets user context
await db.query("SET LOCAL app.user_id = $1", [currentUserId]);
await db.query("UPDATE users SET email = $1 WHERE id = $2", [newEmail, userId]);
// Audit log automatically created
```

### Best Practices

- **Partition by time**: Essential for time-series. Makes dropping old data instant and keeps active partitions small and fast.
- **Minimal indexes**: Only index time and frequently filtered columns. Too many indexes slow down high-volume inserts.
- **Use JSONB for flexible schema**: Tags, metadata, and nested values work well in JSONB. Index with GIN when needed for querying.
- **Batch inserts when possible**: Insert multiple rows in single query for better performance. Use COPY for bulk loading.
- **Consider TimescaleDB**: For very high-volume time-series (millions of rows/day), TimescaleDB extension adds compression and better query performance.
- **Immutable audit logs**: Never update or delete audit logs. Ensure triggers cover all relevant tables and actions.

---

## Question 5: What's your approach to database migrations and rollback strategies?

### Conceptual Explanation

Database migrations evolve schema over time while maintaining data integrity and zero downtime. Challenges include: coordinating code and schema changes, handling large tables (long-running ALTERs), backward compatibility, and safe rollbacks.

Best practices: **incremental migrations** (one change per migration), **backward-compatible changes** (add column with NULL, populate, then add NOT NULL), **test on production-like data**, and **use migration tools** (node-pg-migrate, knex, TypeORM). Avoid destructive changes (DROP COLUMN) until confirmed all code is updated.

Rollback strategies depend on the change. Adding columns/indexes is easy to reverse. Dropping columns loses data. Use **multi-phase rollouts**: Phase 1 (add new column), Phase 2 (populate data), Phase 3 (update code to use new column), Phase 4 (remove old column).

### Code Example

```javascript
// Using node-pg-migrate
// migrations/1705000000000_add_user_status.js

exports.up = pgm => {
  // Add column with default (safe, doesn't lock table long)
  pgm.addColumn('users', {
    status: {
      type: 'varchar(20)',
      default: 'active',
      notNull: true
    }
  });
  
  // Add index concurrently (doesn't block writes)
  pgm.createIndex('users', 'status', { 
    method: 'btree',
    concurrently: true 
  });
};

exports.down = pgm => {
  pgm.dropColumn('users', 'status');
};

// Complex migration: Renaming column safely (multi-phase)

// Phase 1: Add new column
exports.up = pgm => {
  pgm.addColumn('users', {
    full_name: { type: 'varchar(255)', notNull: false }
  });
};

// Phase 2: Backfill data (run separately, can be batched)
exports.up = pgm => {
  pgm.sql(`
    UPDATE users 
    SET full_name = name 
    WHERE full_name IS NULL
  `);
};

// Phase 3: Make new column NOT NULL (after code supports both columns)
exports.up = pgm => {
  pgm.alterColumn('users', 'full_name', { notNull: true });
};

// Phase 4: Drop old column (after all code uses new column)
exports.up = pgm => {
  pgm.dropColumn('users', 'name');
};

// Large table migration (add column without locking)
exports.up = async pgm => {
  // Add column allowing NULL first (instant)
  pgm.addColumn('orders', {
    tracking_number: { type: 'varchar(50)', notNull: false }
  });
  
  // Backfill in batches (in separate script/job, not in migration)
  // See backfill script below
  
  // Later migration: Add NOT NULL constraint
  // After verifying all rows populated
};

// Backfill script (run separately)
async function backfillTrackingNumbers() {
  let lastId = 0;
  const batchSize = 1000;
  
  while (true) {
    const result = await db.query(`
      UPDATE orders
      SET tracking_number = generate_tracking_number(id)
      WHERE id > $1 AND tracking_number IS NULL
      ORDER BY id
      LIMIT $2
      RETURNING id
    `, [lastId, batchSize]);
    
    if (result.rows.length === 0) break;
    
    lastId = result.rows[result.rows.length - 1].id;
    console.log(`Backfilled up to ID ${lastId}`);
    
    // Pause to avoid overloading DB
    await sleep(100);
  }
}

// Migration with data transformation
exports.up = pgm => {
  // Add JSONB column
  pgm.addColumn('users', {
    preferences: { type: 'jsonb', default: '{}' }
  });
  
  // Migrate data from old columns
  pgm.sql(`
    UPDATE users
    SET preferences = jsonb_build_object(
      'theme', theme,
      'language', language,
      'notifications', notify_enabled
    )
  `);
  
  // Drop old columns (in later migration, after verification)
};

// Testing migrations
describe('Migration: add user status', () => {
  it('should add status column with default', async () => {
    await runMigration('1705000000000_add_user_status');
    
    const result = await db.query(`
      SELECT column_name, column_default, is_nullable
      FROM information_schema.columns
      WHERE table_name = 'users' AND column_name = 'status'
    `);
    
    expect(result.rows[0].column_default).toBe("'active'::character varying");
    expect(result.rows[0].is_nullable).toBe('NO');
  });
  
  it('should rollback cleanly', async () => {
    await runMigration('1705000000000_add_user_status');
    await rollbackMigration('1705000000000_add_user_status');
    
    const result = await db.query(`
      SELECT column_name FROM information_schema.columns
      WHERE table_name = 'users' AND column_name = 'status'
    `);
    
    expect(result.rows.length).toBe(0);
  });
});
```

### Best Practices

- **One logical change per migration**: Easier to understand, test, and rollback. Keep migrations focused and small.
- **Always write down() method**: Even if you "never rollback", having the rollback code documents what the migration does.
- **Use CREATE INDEX CONCURRENTLY**: Prevents table locks on large tables. Regular CREATE INDEX blocks all writes until complete.
- **Test on production-sized data**: Migrations fast on dev data (1000 rows) may timeout on production (10M rows).
- **Backward-compatible changes**: New code should work with old schema for brief period during deployment. Enables zero-downtime deploys.
- **Use transactions wisely**: Most DDL is transactional in Postgres, but CREATE INDEX CONCURRENTLY can't run in transaction.
- **Monitor long-running migrations**: For large tables, run migrations during low-traffic periods. Monitor locks and query performance.
- **Keep old columns temporarily**: After adding new column and migrating code, keep old column for a deployment cycle as safety net.

