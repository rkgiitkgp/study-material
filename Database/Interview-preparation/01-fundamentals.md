# I. Fundamentals (All Databases)

**Goal:** Test conceptual clarity and basic understanding.

---

## A. Core Concepts

### 1. What is a database? Difference between DBMS and RDBMS?

**Database:** A database is an organized collection of structured data stored electronically in a computer system, designed to be easily accessed, managed, and updated.

**DBMS (Database Management System):**
- Software that manages databases
- Stores data in files or hierarchical form
- No relationship between data
- Examples: File systems, XML databases

**RDBMS (Relational Database Management System):**
- Stores data in tables with relationships between them
- Uses SQL for data manipulation
- Enforces data integrity through constraints (Primary Key, Foreign Key)
- Follows ACID properties
- Examples: PostgreSQL, MySQL, Oracle, SQL Server

**Key Differences:**

| Feature | DBMS | RDBMS |
|---------|------|-------|
| Data Storage | Files | Tables (rows & columns) |
| Relationships | No relationships | Foreign keys, relationships |
| Normalization | Not supported | Supported |
| Users | Single user | Multiple concurrent users |
| Examples | XML, File systems | PostgreSQL, MySQL |

---

### 2. What are ACID properties? Explain each.

ACID properties ensure reliable transaction processing in databases:

**A - Atomicity:**
- Transactions are "all or nothing"
- Either all operations complete successfully or none do
- Example: Bank transfer

```sql
BEGIN TRANSACTION;
UPDATE accounts SET balance = balance - 100 WHERE account_id = 1; -- Debit
UPDATE accounts SET balance = balance + 100 WHERE account_id = 2; -- Credit
COMMIT; -- Both succeed
-- If any operation fails, ROLLBACK happens and no changes are made
```

**C - Consistency:**
- Database moves from one valid state to another
- All rules, constraints, and triggers are maintained
- Example: If a constraint says balance cannot be negative, the transaction will fail if it tries to make balance negative

**I - Isolation:**
- Concurrent transactions don't interfere with each other
- Intermediate states are invisible to other transactions
- Example: Two users booking the same seat simultaneously - isolation ensures only one succeeds

```sql
-- Transaction 1
BEGIN;
SELECT status FROM seats WHERE seat_id = 10; -- available
UPDATE seats SET status = 'booked' WHERE seat_id = 10;
COMMIT;

-- Transaction 2 (concurrent) won't see Transaction 1's changes until commit
```

**D - Durability:**
- Once committed, changes persist even after system failures
- Changes are written to non-volatile storage
- Example: After confirming payment, power failure won't lose the transaction - it's permanently saved

---

### 3. Difference between OLTP and OLAP.

**OLTP (Online Transaction Processing):**
- Handles day-to-day transactional operations
- Focused on INSERT, UPDATE, DELETE operations
- Supports many concurrent users
- Fast query response time
- Normalized data structure
- Examples: Banking systems, E-commerce orders, Booking systems

**OLAP (Online Analytical Processing):**
- Handles complex analytical queries
- Focused on SELECT operations for reporting and analysis
- Fewer concurrent users
- Longer query execution time
- Denormalized/dimensional data (Star/Snowflake schema)
- Examples: Business Intelligence, Data Warehouses, Reporting systems

**Comparison Table:**

| Aspect | OLTP | OLAP |
|--------|------|------|
| Purpose | Daily transactions | Data analysis & reporting |
| Operations | INSERT, UPDATE, DELETE | Complex SELECT queries |
| Data Volume | Gigabytes | Terabytes to Petabytes |
| Query Type | Simple, fast queries | Complex aggregations |
| Users | Thousands | Hundreds |
| Response Time | Milliseconds | Seconds to minutes |
| Database Design | Normalized (3NF) | Denormalized (Star schema) |
| Examples | PostgreSQL, MySQL | Snowflake, Redshift, BigQuery |

**Example Scenarios:**

OLTP Query:
```sql
-- Add new order
INSERT INTO orders (customer_id, product_id, quantity, price)
VALUES (101, 505, 2, 29.99);
```

OLAP Query:
```sql
-- Analyze sales trends by region and product category over time
SELECT 
    region,
    product_category,
    DATE_TRUNC('month', order_date) as month,
    SUM(quantity * price) as total_revenue,
    AVG(price) as avg_price,
    COUNT(DISTINCT customer_id) as unique_customers
FROM fact_sales
JOIN dim_product ON fact_sales.product_id = dim_product.id
JOIN dim_location ON fact_sales.location_id = dim_location.id
WHERE order_date >= '2024-01-01'
GROUP BY region, product_category, month
ORDER BY month, total_revenue DESC;
```

---

### 4. Explain normalization and denormalization.

**Normalization:**  
Process of organizing data to minimize redundancy and dependency by dividing large tables into smaller ones and defining relationships.

**Normal Forms:**

**1NF (First Normal Form):**
- Each column contains atomic (indivisible) values
- Each column contains values of a single type
- Each column has a unique name

Before 1NF:
```
| student_id | name  | courses              |
|------------|-------|---------------------|
| 1          | John  | Math, Physics       |
```

After 1NF:
```
| student_id | name  | course   |
|------------|-------|----------|
| 1          | John  | Math     |
| 1          | John  | Physics  |
```

**2NF (Second Normal Form):**
- Must be in 1NF
- No partial dependency (non-key attributes depend on entire primary key)

Before 2NF:
```
| student_id | course_id | student_name | course_name |
|------------|-----------|--------------|-------------|
| 1          | 101       | John         | Math        |
```
Problem: student_name depends only on student_id, not the composite key

After 2NF:
```
Students table:
| student_id | student_name |
|------------|--------------|
| 1          | John         |

Enrollments table:
| student_id | course_id |
|------------|-----------|
| 1          | 101       |

Courses table:
| course_id | course_name |
|-----------|-------------|
| 101       | Math        |
```

**3NF (Third Normal Form):**
- Must be in 2NF
- No transitive dependency (non-key attributes depend only on primary key)

Before 3NF:
```
| student_id | name | zip_code | city     |
|------------|------|----------|----------|
| 1          | John | 10001    | New York |
```
Problem: city depends on zip_code, not directly on student_id

After 3NF:
```
Students table:
| student_id | name | zip_code |
|------------|------|----------|
| 1          | John | 10001    |

Zip_codes table:
| zip_code | city     |
|----------|----------|
| 10001    | New York |
```

**Denormalization:**  
Process of intentionally introducing redundancy to improve read performance by combining tables.

**When to Denormalize:**
- Read-heavy applications
- Need to avoid expensive JOINs
- Acceptable to have some data redundancy
- OLAP systems

**Example:**
```sql
-- Normalized (requires JOIN)
SELECT o.order_id, c.customer_name, c.email
FROM orders o
JOIN customers c ON o.customer_id = c.id;

-- Denormalized (no JOIN needed)
SELECT order_id, customer_name, customer_email
FROM orders;
-- customer_name and customer_email stored directly in orders table
```

**Trade-offs:**

| Aspect | Normalization | Denormalization |
|--------|---------------|-----------------|
| Redundancy | Minimal | High |
| Storage | Less space | More space |
| Write Speed | Faster | Slower (update multiple places) |
| Read Speed | Slower (JOINs) | Faster (no JOINs) |
| Data Integrity | Easier | Harder |
| Use Case | OLTP | OLAP |

---

### 5. Types of indexes and their use cases.

Indexes are data structures that improve the speed of data retrieval operations on database tables.

**1. B-Tree Index (Default in most databases):**
- Balanced tree structure
- Good for equality and range queries
- Most commonly used index type

**Use Cases:**
- Primary keys
- Columns used in WHERE, ORDER BY, GROUP BY
- Foreign key columns

**Example:**
```sql
CREATE INDEX idx_user_email ON users(email);

-- Query that benefits
SELECT * FROM users WHERE email = 'user@example.com';
SELECT * FROM users WHERE age BETWEEN 25 AND 35;
```

**2. Hash Index:**
- Uses hash function to map keys to values
- Very fast for equality comparisons
- Cannot be used for range queries

**Use Cases:**
- Exact match lookups
- Cache key lookups

**Example:**
```sql
CREATE INDEX idx_session_hash ON sessions USING HASH(session_id);

-- Query that benefits
SELECT * FROM sessions WHERE session_id = 'abc123';
```

**3. Bitmap Index:**
- Uses bitmaps (0s and 1s) for each distinct value
- Efficient for low-cardinality columns
- Great for data warehouses

**Use Cases:**
- Boolean columns
- Status fields (active/inactive)
- Gender columns

**Example:**
```sql
-- Conceptual (syntax varies by database)
CREATE BITMAP INDEX idx_order_status ON orders(status);

-- Query that benefits
SELECT * FROM orders WHERE status = 'SHIPPED' AND priority = 'HIGH';
```

**4. Full-Text Index:**
- Specialized for text search
- Supports natural language queries
- Handles word stemming, relevance ranking

**Use Cases:**
- Search engines
- Document search
- Product descriptions

**Example:**
```sql
-- PostgreSQL
CREATE INDEX idx_product_description ON products 
USING GIN(to_tsvector('english', description));

-- Query that benefits
SELECT * FROM products 
WHERE to_tsvector('english', description) @@ to_tsquery('laptop & wireless');
```

**5. Composite (Multi-Column) Index:**
- Index on multiple columns
- Order of columns matters
- Good for queries filtering on multiple columns

**Use Cases:**
- Multi-column WHERE clauses
- Covering indexes

**Example:**
```sql
CREATE INDEX idx_user_location ON users(country, state, city);

-- Query that benefits (uses index)
SELECT * FROM users WHERE country = 'USA' AND state = 'CA' AND city = 'LA';

-- Partial benefit (uses index for country only)
SELECT * FROM users WHERE country = 'USA';

-- No benefit (doesn't start with first column)
SELECT * FROM users WHERE city = 'LA';
```

**6. Unique Index:**
- Ensures uniqueness of column values
- Automatically created for PRIMARY KEY and UNIQUE constraints

**Example:**
```sql
CREATE UNIQUE INDEX idx_user_username ON users(username);

-- Prevents duplicate usernames
INSERT INTO users(username) VALUES ('john_doe'); -- OK
INSERT INTO users(username) VALUES ('john_doe'); -- ERROR: duplicate
```

**7. Partial Index:**
- Index on subset of rows based on condition
- Smaller, faster index
- PostgreSQL specific feature

**Example:**
```sql
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;

-- Query that benefits
SELECT * FROM users WHERE email = 'user@example.com' AND is_active = true;
```

**8. Covering Index (Include Columns):**
- Includes non-key columns in index
- Query can be satisfied entirely from index without accessing table

**Example:**
```sql
CREATE INDEX idx_orders_covering ON orders(customer_id) 
INCLUDE (order_date, total_amount);

-- Query satisfied entirely from index (Index-Only Scan)
SELECT order_date, total_amount FROM orders WHERE customer_id = 123;
```

**Index Trade-offs:**

| Pros | Cons |
|------|------|
| Faster SELECT queries | Slower INSERT/UPDATE/DELETE |
| Faster sorting (ORDER BY) | Requires storage space |
| Faster grouping (GROUP BY) | Requires maintenance |
| Enforce uniqueness | Can be outdated (needs rebuild) |

**Best Practices:**
- Don't over-index (every index slows writes)
- Index columns used frequently in WHERE clauses
- Consider composite indexes for multi-column queries
- Monitor and remove unused indexes
- Rebuild indexes periodically to prevent fragmentation

---

### 6. What is a transaction? How does rollback and commit work?

**Transaction:**  
A transaction is a sequence of one or more SQL operations treated as a single logical unit of work. All operations either complete successfully (commit) or fail together (rollback), ensuring data consistency.

**Transaction Lifecycle:**

1. **BEGIN/START TRANSACTION** - Starts a transaction
2. **Execute SQL operations** - Perform database operations
3. **COMMIT** - Save all changes permanently  
   OR  
   **ROLLBACK** - Undo all changes

**How COMMIT works:**
- Writes all changes from transaction log to database
- Makes changes visible to other users
- Releases locks held by transaction
- Changes are permanent and survive system crashes

**How ROLLBACK works:**
- Discards all changes made during transaction
- Returns database to state before transaction began
- Releases locks
- Can be triggered automatically on errors

**Example 1: Successful Transaction with COMMIT**
```sql
BEGIN TRANSACTION;

-- Transfer $500 from Account A to Account B
UPDATE accounts SET balance = balance - 500 WHERE account_id = 'A';
UPDATE accounts SET balance = balance + 500 WHERE account_id = 'B';

-- Both operations succeeded, save changes
COMMIT;

-- Changes are now permanent
```

**Example 2: Failed Transaction with ROLLBACK**
```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 500 WHERE account_id = 'A';  -- Success

-- Try to credit Account B, but it doesn't exist or has constraint violation
UPDATE accounts SET balance = balance + 500 WHERE account_id = 'B';  -- ERROR!

-- Undo all changes (including the first UPDATE)
ROLLBACK;

-- Database is back to original state, no money lost
```

**Example 3: Conditional ROLLBACK**
```sql
BEGIN TRANSACTION;

UPDATE accounts SET balance = balance - 1000 WHERE account_id = 'A';

-- Check if balance went negative
SELECT balance INTO @new_balance FROM accounts WHERE account_id = 'A';

IF @new_balance < 0 THEN
    ROLLBACK;  -- Insufficient funds, cancel transaction
    SELECT 'Transaction failed: Insufficient funds' AS message;
ELSE
    COMMIT;  -- Sufficient funds, complete transaction
    SELECT 'Transaction successful' AS message;
END IF;
```

**Example 4: Savepoints (Partial Rollback)**
```sql
BEGIN TRANSACTION;

INSERT INTO orders (customer_id, total) VALUES (1, 100);
SAVEPOINT order_created;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 101, 2);
SAVEPOINT items_added;

INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 999, 1);  -- Invalid product!

-- Rollback only to items_added savepoint, keep the order
ROLLBACK TO SAVEPOINT items_added;

-- Add correct item
INSERT INTO order_items (order_id, product_id, quantity) VALUES (1, 102, 1);

COMMIT;  -- Saves order and the correct items
```

**Auto-commit Mode:**  
Most databases have auto-commit enabled by default:
```sql
-- Each statement is automatically committed
INSERT INTO users (name) VALUES ('John');  -- Auto-committed
UPDATE users SET name = 'Jane' WHERE id = 1;  -- Auto-committed

-- Disable auto-commit for manual transaction control
SET autocommit = 0;  -- MySQL
-- or
BEGIN TRANSACTION;  -- Explicitly start transaction
```

**Transaction Isolation and Concurrent Access:**
```sql
-- Session 1
BEGIN TRANSACTION;
UPDATE products SET stock = stock - 1 WHERE id = 100;
-- Changes not visible to other sessions yet
COMMIT;  -- Now changes are visible to all

-- Session 2 (concurrent)
BEGIN TRANSACTION;
SELECT stock FROM products WHERE id = 100;
-- Sees old value until Session 1 commits
```

**Real-World Use Case: E-commerce Order**
```sql
BEGIN TRANSACTION;

-- 1. Create order
INSERT INTO orders (customer_id, order_date, total) 
VALUES (123, NOW(), 199.99);

SET @order_id = LAST_INSERT_ID();

-- 2. Add order items
INSERT INTO order_items (order_id, product_id, quantity, price)
VALUES (@order_id, 456, 2, 99.99);

-- 3. Update inventory
UPDATE products SET stock = stock - 2 WHERE product_id = 456;

-- 4. Check if stock went negative
SELECT stock INTO @current_stock FROM products WHERE product_id = 456;

IF @current_stock < 0 THEN
    ROLLBACK;  -- Not enough stock, cancel entire order
ELSE
    -- 5. Process payment (external API call simulation)
    -- If payment fails, ROLLBACK
    -- If payment succeeds, COMMIT
    COMMIT;
END IF;
```

**Key Points:**
- Transactions ensure ACID properties
- COMMIT makes changes permanent
- ROLLBACK undoes all changes
- Use savepoints for partial rollbacks
- Always wrap related operations in a transaction
- Handle errors and rollback appropriately

---

### 7. What is a deadlock, and how do you prevent it?

**Deadlock:**  
A deadlock occurs when two or more transactions are waiting for each other to release locks, creating a circular dependency where none can proceed.

**Classic Deadlock Scenario:**

```
Time    Transaction 1                           Transaction 2
----    ----------------------------------      ----------------------------------
T1      BEGIN;                                  BEGIN;
T2      UPDATE accounts SET balance = 1000      
        WHERE id = 1;                           
        (Locks Row 1)                           
T3                                              UPDATE accounts SET balance = 2000
                                                WHERE id = 2;
                                                (Locks Row 2)
T4      UPDATE accounts SET balance = 1500      
        WHERE id = 2;                           
        (Waits for Row 2 lock)                  
T5                                              UPDATE accounts SET balance = 2500
                                                WHERE id = 1;
                                                (Waits for Row 1 lock)
        
        🔒 DEADLOCK! Each transaction waits for the other
```

**Real Example:**
```sql
-- Transaction 1
BEGIN;
UPDATE orders SET status = 'processing' WHERE order_id = 100;  -- Locks order 100
-- ... some processing ...
UPDATE inventory SET stock = stock - 1 WHERE product_id = 500;  -- Waits for inventory lock

-- Transaction 2 (running concurrently)
BEGIN;
UPDATE inventory SET stock = stock - 1 WHERE product_id = 500;  -- Locks inventory 500
-- ... some processing ...
UPDATE orders SET status = 'processing' WHERE order_id = 100;  -- Waits for order lock

-- DEADLOCK DETECTED!
-- Database will automatically abort one transaction
```

**How Databases Handle Deadlocks:**
1. **Detection:** Database periodically checks for circular wait conditions
2. **Victim Selection:** Chooses one transaction to abort (usually the one with least work done)
3. **Rollback:** Aborts victim transaction and releases its locks
4. **Retry:** Application should catch error and retry the transaction

**Deadlock Prevention Strategies:**

**1. Lock Resources in Consistent Order**
```sql
-- ❌ Bad: Different lock order
-- Transaction 1
UPDATE table_A WHERE id = 1;
UPDATE table_B WHERE id = 2;

-- Transaction 2
UPDATE table_B WHERE id = 2;  -- Different order!
UPDATE table_A WHERE id = 1;

-- ✅ Good: Same lock order
-- Transaction 1
UPDATE table_A WHERE id = 1;
UPDATE table_B WHERE id = 2;

-- Transaction 2
UPDATE table_A WHERE id = 1;  -- Same order
UPDATE table_B WHERE id = 2;
```

**2. Keep Transactions Short**
```sql
-- ❌ Bad: Long transaction with external calls
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
-- Call external API (holds lock for long time)
COMMIT;

-- ✅ Good: Short transaction
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
COMMIT;
-- Then call external API
```

**3. Use Lower Isolation Levels (When Appropriate)**
```sql
-- ❌ SERIALIZABLE: Highest isolation, more locks, more deadlocks
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;

-- ✅ READ COMMITTED: Lower isolation, fewer locks
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
```

**4. Use SELECT FOR UPDATE with NOWAIT or SKIP LOCKED**
```sql
-- ❌ Regular SELECT then UPDATE (can cause deadlock)
BEGIN;
SELECT * FROM orders WHERE id = 100;
-- ... processing ...
UPDATE orders SET status = 'shipped' WHERE id = 100;
COMMIT;

-- ✅ Lock immediately or fail fast
BEGIN;
SELECT * FROM orders WHERE id = 100 FOR UPDATE NOWAIT;  -- Fails if locked
-- or
SELECT * FROM orders WHERE id = 100 FOR UPDATE SKIP LOCKED;  -- Skips locked rows
UPDATE orders SET status = 'shipped' WHERE id = 100;
COMMIT;
```

**5. Use Optimistic Locking**
```sql
-- Add version column to table
ALTER TABLE accounts ADD COLUMN version INT DEFAULT 0;

-- Update with version check
UPDATE accounts 
SET balance = balance - 100, 
    version = version + 1
WHERE id = 1 
  AND version = 5;  -- Only update if version hasn't changed

-- If no rows updated, someone else modified it - retry
```

**6. Set Lock Timeout**
```sql
-- PostgreSQL
SET lock_timeout = '5s';

-- MySQL
SET innodb_lock_wait_timeout = 5;

-- Transaction will abort if it waits more than 5 seconds
```

**7. Avoid User Interaction During Transactions**
```sql
-- ❌ Bad: Waiting for user input while holding locks
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 100;
-- Wait for user to confirm shipping address... (holds lock!)
COMMIT;

-- ✅ Good: Get all user input first
-- Get user confirmation
BEGIN;
UPDATE products SET stock = stock - 1 WHERE id = 100;
COMMIT;  -- Release lock quickly
```

**8. Use Database-Level Deadlock Detection**
```sql
-- PostgreSQL: View current locks
SELECT * FROM pg_locks WHERE NOT granted;

-- MySQL: Show current transactions
SELECT * FROM information_schema.innodb_trx;

-- Check for deadlocks in logs
SHOW ENGINE INNODB STATUS;  -- MySQL
```

**Handling Deadlocks in Application Code:**
```python
# Python example with retry logic
import psycopg2
from time import sleep

def transfer_money(from_account, to_account, amount, max_retries=3):
    for attempt in range(max_retries):
        try:
            conn = psycopg2.connect(...)
            cursor = conn.cursor()
            
            cursor.execute("BEGIN")
            
            # Lock in consistent order (lower ID first)
            if from_account < to_account:
                cursor.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", 
                             (amount, from_account))
                cursor.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", 
                             (amount, to_account))
            else:
                cursor.execute("UPDATE accounts SET balance = balance + %s WHERE id = %s", 
                             (amount, to_account))
                cursor.execute("UPDATE accounts SET balance = balance - %s WHERE id = %s", 
                             (amount, from_account))
            
            cursor.execute("COMMIT")
            return True
            
        except psycopg2.extensions.TransactionRollbackError as e:
            # Deadlock detected
            print(f"Deadlock detected, retrying... (attempt {attempt + 1})")
            sleep(0.1 * (2 ** attempt))  # Exponential backoff
            
        except Exception as e:
            cursor.execute("ROLLBACK")
            raise
    
    return False  # Failed after max retries
```

**Best Practices Summary:**
1. ✅ Access tables/rows in same order across all transactions
2. ✅ Keep transactions as short as possible
3. ✅ Use appropriate isolation levels
4. ✅ Implement retry logic in application
5. ✅ Use timeouts to prevent indefinite waits
6. ✅ Monitor deadlock frequency in production
7. ✅ Avoid holding locks during external calls or user input
8. ✅ Consider optimistic locking for high contention scenarios

---

## B. Query Optimization Basics

### 1. What is an execution plan?

**Execution Plan (Query Plan):**  
A detailed roadmap showing how the database engine will execute a SQL query, including which indexes to use, join methods, and the order of operations.

**Why It's Important:**
- Shows actual cost of query execution
- Identifies performance bottlenecks
- Helps optimize slow queries
- Reveals if indexes are being used

**Components of an Execution Plan:**
1. **Scan Type:** Sequential scan, Index scan, Index-only scan
2. **Join Method:** Nested Loop, Hash Join, Merge Join
3. **Cost Estimates:** Startup cost, total cost, rows returned
4. **Execution Time:** Planning time, execution time

**How to View Execution Plans:**

**PostgreSQL:**
```sql
-- Show execution plan without running query
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Show execution plan WITH actual runtime statistics
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';

-- More detailed output
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) 
SELECT * FROM orders WHERE order_date > '2024-01-01';
```

**MySQL:**
```sql
-- Show execution plan
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';

-- Detailed format
EXPLAIN FORMAT=JSON SELECT * FROM users WHERE email = 'user@example.com';

-- Analyze with actual execution
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
```

**Example 1: Query Without Index**
```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';

-- Output:
-- Seq Scan on users  (cost=0.00..1693.00 rows=1 width=128) 
--   (actual time=0.234..12.456 rows=1 loops=1)
--   Filter: (email = 'john@example.com')
--   Rows Removed by Filter: 99999
-- Planning Time: 0.123 ms
-- Execution Time: 12.567 ms

-- Problem: Sequential scan = checking every row (slow!)
```

**Example 2: Same Query With Index**
```sql
-- Create index
CREATE INDEX idx_users_email ON users(email);

EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'john@example.com';

-- Output:
-- Index Scan using idx_users_email on users  (cost=0.29..8.31 rows=1 width=128) 
--   (actual time=0.045..0.047 rows=1 loops=1)
--   Index Cond: (email = 'john@example.com')
-- Planning Time: 0.234 ms
-- Execution Time: 0.089 ms

-- Much faster! Direct lookup using index
```

**Reading Execution Plans:**

**Key Terms:**
- **Seq Scan** (Sequential Scan): Reads entire table row by row - SLOW for large tables
- **Index Scan**: Uses index to find rows - FAST
- **Index Only Scan**: Data retrieved entirely from index without accessing table - FASTEST
- **Bitmap Heap Scan**: Uses multiple indexes combined with bitmap
- **Nested Loop**: Join method - good for small datasets
- **Hash Join**: Join method - good for large datasets
- **Merge Join**: Join method - good for sorted data

**Example 3: Complex Query with Joins**
```sql
EXPLAIN ANALYZE
SELECT 
    o.order_id,
    c.customer_name,
    p.product_name,
    oi.quantity
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= '2024-01-01'
  AND c.country = 'USA';

-- Output might show:
-- Hash Join  (cost=1234.56..5678.90 rows=500 width=64)
--   Hash Cond: (o.customer_id = c.customer_id)
--   ->  Seq Scan on orders o  (cost=0.00..234.00 rows=1000 width=16)
--         Filter: (order_date >= '2024-01-01')
--   ->  Hash  (cost=987.00..987.00 rows=2000 width=48)
--         ->  Index Scan using idx_customers_country on customers c  
--             (cost=0.29..987.00 rows=2000 width=48)
--               Index Cond: (country = 'USA')
-- ...
```

**Interpreting Costs:**
```sql
-- Format: (cost=startup_cost..total_cost rows=estimated_rows width=avg_row_bytes)

-- Low cost (good):
Index Scan (cost=0.29..8.31 rows=1 width=128)

-- High cost (bad):
Seq Scan (cost=0.00..50000.00 rows=1000000 width=128)
```

**Common Performance Issues Revealed:**
1. **Sequential Scans on Large Tables**  
   Solution: Add appropriate indexes

2. **Nested Loop Joins on Large Tables**  
   Solution: Increase `work_mem`, ensure proper indexes

3. **High Filter Removal Count**  
   Solution: Add WHERE clause conditions to index

4. **Missing Index Usage**  
   Solution: Create index on frequently filtered columns

**Example 4: Optimizing Based on Execution Plan**
```sql
-- BEFORE: Slow query
EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE EXTRACT(YEAR FROM order_date) = 2024;

-- Output: Seq Scan (function on column prevents index usage)

-- AFTER: Optimized query
CREATE INDEX idx_orders_date ON orders(order_date);

EXPLAIN ANALYZE 
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2025-01-01';

-- Output: Index Scan (much faster!)
```

**Practical Tips:**
1. Always use `EXPLAIN ANALYZE` to see actual execution
2. Look for sequential scans on large tables
3. Check if indexes are being used
4. Compare estimated vs actual rows (large difference = outdated statistics)
5. Run `ANALYZE` to update table statistics
6. Watch for "Rows Removed by Filter" (indicates inefficient filtering)

**Updating Statistics:**
```sql
-- PostgreSQL
ANALYZE users;  -- Update statistics for one table
ANALYZE;        -- Update statistics for all tables

-- MySQL
ANALYZE TABLE users;
```

---

### 2. How does the optimizer choose indexes?

The query optimizer is a component of the database that determines the most efficient way to execute a query by evaluating different execution plans and choosing the one with the lowest estimated cost.

**Factors the Optimizer Considers When Choosing Indexes:**

**1. Table Statistics**
- Number of rows in table
- Distribution of values (cardinality)
- Data distribution histograms
- Index size and depth

**2. Selectivity**
- How many rows will be filtered
- High selectivity (few rows) → Use index
- Low selectivity (many rows) → Might skip index

**3. Index Coverage**
- Whether index contains all columns needed
- Covering indexes avoid table lookups

**4. Query Predicates**
- Columns in WHERE clause
- Columns in JOIN conditions
- Columns in ORDER BY

**5. Cost Estimation**
- I/O cost (disk reads)
- CPU cost (processing)
- Memory available
- Index vs sequential scan cost

**How the Optimizer Decides:**

**Example 1: High Selectivity - Uses Index**
```sql
-- Table: 1,000,000 users
-- Query returns 1 user (0.0001% of rows)
SELECT * FROM users WHERE user_id = 12345;

-- Optimizer Decision: Use index on user_id
-- Reasoning: Only 1 row to fetch, index lookup is cheap
-- Cost: ~5-10 page reads

EXPLAIN:
Index Scan using idx_user_id on users (cost=0.42..8.44 rows=1)
```

**Example 2: Low Selectivity - Skips Index**
```sql
-- Table: 1,000,000 users
-- Query returns 800,000 users (80% of rows)
SELECT * FROM users WHERE is_active = true;

-- Optimizer Decision: Sequential scan (ignore index)
-- Reasoning: Need to read most of table anyway
-- Cost with index: Read index (1M entries) + Random table lookups (800K) = Very expensive
-- Cost with seq scan: Read entire table sequentially = Cheaper

EXPLAIN:
Seq Scan on users (cost=0.00..18334.00 rows=800000)
  Filter: (is_active = true)
```

**Example 3: Multiple Indexes Available**
```sql
-- Indexes available:
-- 1. idx_email on (email)
-- 2. idx_country on (country)
-- 3. idx_created on (created_at)

SELECT * FROM users 
WHERE email = 'john@example.com' 
  AND country = 'USA'
  AND created_at > '2024-01-01';

-- Optimizer evaluates:
-- Option 1: Use idx_email (most selective, email is unique)
--   Cost: 1 row to fetch = CHEAPEST
-- Option 2: Use idx_country (less selective)
--   Cost: ~10,000 rows to filter = More expensive
-- Option 3: Use idx_created (less selective)
--   Cost: ~50,000 rows to filter = Most expensive

-- Decision: Use idx_email (lowest cost)

EXPLAIN:
Index Scan using idx_email on users (cost=0.42..8.44 rows=1)
  Filter: (country = 'USA' AND created_at > '2024-01-01')
```

**Example 4: Composite Index vs Multiple Single Indexes**
```sql
-- Scenario A: Composite index
CREATE INDEX idx_user_location ON users(country, state, city);

SELECT * FROM users 
WHERE country = 'USA' 
  AND state = 'CA' 
  AND city = 'Los Angeles';

-- Optimizer Decision: Use composite index
-- Very efficient - can filter on all three columns in one index lookup

EXPLAIN:
Index Scan using idx_user_location on users (cost=0.42..12.45 rows=5)
  Index Cond: (country = 'USA' AND state = 'CA' AND city = 'Los Angeles')


-- Scenario B: Only single-column indexes
CREATE INDEX idx_country ON users(country);
CREATE INDEX idx_state ON users(state);
CREATE INDEX idx_city ON users(city);

SELECT * FROM users 
WHERE country = 'USA' 
  AND state = 'CA' 
  AND city = 'Los Angeles';

-- Optimizer might:
-- 1. Use idx_country and filter rest in memory, OR
-- 2. Use Bitmap Index Scan (combine multiple indexes), OR
-- 3. Choose most selective single index

EXPLAIN:
Bitmap Heap Scan on users (cost=23.45..456.78 rows=5)
  Recheck Cond: (country = 'USA' AND state = 'CA')
  ->  BitmapAnd
        ->  Bitmap Index Scan on idx_country
        ->  Bitmap Index Scan on idx_state
  Filter: (city = 'Los Angeles')
```

**Example 5: Index Not Used Due to Function on Column**
```sql
-- Index exists but NOT used
CREATE INDEX idx_email ON users(email);

-- ❌ Function on indexed column prevents index usage
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

EXPLAIN:
Seq Scan on users (cost=0.00..18334.00 rows=5000)
  Filter: (LOWER(email) = 'john@example.com')


-- ✅ Solution 1: Remove function
SELECT * FROM users WHERE email = 'john@example.com';

-- ✅ Solution 2: Create functional/expression index
CREATE INDEX idx_email_lower ON users(LOWER(email));
SELECT * FROM users WHERE LOWER(email) = 'john@example.com';

EXPLAIN:
Index Scan using idx_email_lower on users (cost=0.42..8.44 rows=1)
  Index Cond: (LOWER(email) = 'john@example.com')
```

**Selectivity Calculation Example:**
```sql
-- Table statistics
Total rows: 1,000,000
Distinct countries: 50

-- Query: SELECT * FROM users WHERE country = 'USA'
Expected rows = 1,000,000 / 50 = 20,000 rows (2% selectivity)

-- Optimizer calculation:
-- Index scan cost = Index lookup (5 pages) + Fetch 20,000 rows (random I/O)
--                 ≈ 5 + 20,000 = 20,005 page reads
-- Sequential scan = Read entire table sequentially
--                 ≈ 10,000 page reads

-- Decision: Sequential scan is cheaper! Don't use index.
```

**When Optimizer Chooses NOT to Use Index:**

1. **Low Selectivity** (returns large % of rows)
2. **Small Tables** (faster to scan entire table)
3. **Outdated Statistics** (run ANALYZE)
4. **Functions on Indexed Columns**
5. **Implicit Type Conversions**
6. **OR Conditions** (sometimes)

**Example 6: Forcing Index Usage (Not Recommended)**
```sql
-- PostgreSQL: Disable sequential scans (for testing only!)
SET enable_seqscan = OFF;

-- MySQL: Force index usage
SELECT * FROM users FORCE INDEX (idx_email) WHERE email = 'john@example.com';

-- Better approach: Fix the query or update statistics
ANALYZE users;
```

**Best Practices to Help Optimizer:**

**1. Keep Statistics Updated**
```sql
-- PostgreSQL
ANALYZE users;

-- MySQL
ANALYZE TABLE users;

-- Set auto-analyze (PostgreSQL)
ALTER TABLE users SET (autovacuum_analyze_scale_factor = 0.05);
```

**2. Write Index-Friendly Queries**
```sql
-- ❌ Bad: Functions prevent index usage
WHERE YEAR(created_at) = 2024
WHERE UPPER(name) = 'JOHN'

-- ✅ Good: Index can be used
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
WHERE name = 'JOHN'  -- Assuming case-insensitive collation or functional index
```

**3. Create Appropriate Indexes**
```sql
-- For exact matches
CREATE INDEX idx_email ON users(email);

-- For range queries
CREATE INDEX idx_created ON users(created_at);

-- For multiple columns (order matters!)
CREATE INDEX idx_location ON users(country, state, city);

-- For covering queries
CREATE INDEX idx_orders_covering ON orders(customer_id) 
INCLUDE (order_date, total);
```

**4. Monitor Query Performance**
```sql
-- PostgreSQL: Find unused indexes
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%';

-- MySQL: Check index usage
SELECT * FROM sys.schema_unused_indexes;
```

**Key Takeaways:**
- Optimizer uses statistics and cost estimation
- High selectivity → Use index
- Low selectivity → Sequential scan often cheaper
- Keep statistics updated with ANALYZE
- Write queries that can leverage indexes
- Monitor and remove unused indexes

---

### 3. How does caching improve query performance?

Caching stores frequently accessed data in fast memory (RAM) to avoid expensive disk I/O and repeated computation, dramatically improving query response times.

**Types of Database Caching:**

#### 1. Database Buffer Pool / Shared Buffer Cache

The database keeps frequently accessed data pages in memory.

**How it works:**
- First query: Reads from disk (slow) → Stores in buffer cache
- Subsequent queries: Reads from memory (fast) → 1000x faster

**PostgreSQL Example:**
```sql
-- Configure shared_buffers (PostgreSQL)
-- postgresql.conf
shared_buffers = 4GB  -- 25% of total RAM

-- Check buffer cache hit ratio
SELECT 
    sum(heap_blks_read) as disk_reads,
    sum(heap_blks_hit) as cache_hits,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 as cache_hit_ratio
FROM pg_statio_user_tables;

-- Goal: > 99% cache hit ratio
```

**MySQL Example:**
```sql
-- Configure InnoDB buffer pool
-- my.cnf
innodb_buffer_pool_size = 4G  -- 70-80% of RAM

-- Check buffer pool stats
SHOW STATUS LIKE 'Innodb_buffer_pool%';

-- Calculate hit ratio
-- Hit Ratio = (1 - (Innodb_buffer_pool_reads / Innodb_buffer_pool_read_requests)) * 100
```

**Example Performance Impact:**
```sql
-- First execution (cold cache)
SELECT * FROM users WHERE id = 12345;
-- Time: 45ms (disk read)

-- Second execution (warm cache)
SELECT * FROM users WHERE id = 12345;
-- Time: 0.5ms (memory read) ⚡ 90x faster!
```

#### 2. Query Result Cache

Stores the entire result set of a query.

**Use Cases:**
- Expensive aggregation queries
- Frequently run reports
- Dashboard metrics
- Static/slowly changing data

**Application-Level Caching (Redis/Memcached):**
```python
import redis
import psycopg2
import json

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def get_user_profile(user_id):
    cache_key = f"user_profile:{user_id}"
    
    # Try cache first
    cached_data = redis_client.get(cache_key)
    if cached_data:
        print("Cache HIT")
        return json.loads(cached_data)
    
    # Cache miss - query database
    print("Cache MISS - querying database")
    conn = psycopg2.connect(...)
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (user_id,))
    result = cursor.fetchone()
    
    # Store in cache for 1 hour
    redis_client.setex(cache_key, 3600, json.dumps(result))
    
    return result

# First call: Database query (50ms)
user = get_user_profile(12345)  # Cache MISS

# Subsequent calls: Redis cache (1ms)
user = get_user_profile(12345)  # Cache HIT ⚡ 50x faster!
```

**PostgreSQL Query Result Caching (Limited):**  
PostgreSQL doesn't have built-in query cache (removed in modern versions), but you can use:

```sql
-- Use materialized views for expensive queries
CREATE MATERIALIZED VIEW user_stats_summary AS
SELECT 
    DATE_TRUNC('day', created_at) as date,
    COUNT(*) as user_count,
    COUNT(DISTINCT country) as countries
FROM users
GROUP BY date;

-- Refresh when needed
REFRESH MATERIALIZED VIEW user_stats_summary;

-- Fast queries from cached result
SELECT * FROM user_stats_summary WHERE date = '2024-10-01';
```

#### 3. Index Cache

Indexes are cached in memory for faster lookups.

**Example:**
```sql
-- Without index cache: Disk read for index + Disk read for data
-- With index cache: Memory read for index + Disk read for data
-- With both cached: Memory read for index + Memory read for data ⚡

-- B-tree index nodes cached in memory
CREATE INDEX idx_users_email ON users(email);

-- First lookup: Index from disk
SELECT * FROM users WHERE email = 'john@example.com';  -- 20ms

-- Subsequent lookups: Index from cache
SELECT * FROM users WHERE email = 'jane@example.com';  -- 2ms ⚡
```

#### 4. Prepared Statement Cache

Caches the parsed and optimized query execution plan.

**Without Prepared Statements:**
```sql
-- Each execution: Parse → Optimize → Execute
SELECT * FROM users WHERE id = 1;  -- Parse + Optimize + Execute
SELECT * FROM users WHERE id = 2;  -- Parse + Optimize + Execute (repeated work!)
```

**With Prepared Statements:**
```sql
-- First execution: Parse → Optimize → Cache plan
PREPARE user_query AS SELECT * FROM users WHERE id = $1;

-- Subsequent executions: Use cached plan (skip parse/optimize)
EXECUTE user_query(1);  -- Use cached plan ⚡
EXECUTE user_query(2);  -- Use cached plan ⚡
EXECUTE user_query(3);  -- Use cached plan ⚡

DEALLOCATE user_query;
```

**Application Example (Python):**
```python
import psycopg2

conn = psycopg2.connect(...)
cursor = conn.cursor()

# Prepare once
cursor.execute("PREPARE user_query AS SELECT * FROM users WHERE id = $1")

# Execute many times (uses cached plan)
for user_id in range(1, 10000):
    cursor.execute("EXECUTE user_query(%s)", (user_id,))
    # Faster execution due to plan caching
```

#### 5. Connection Pool Caching

Reuses database connections instead of creating new ones.

**Without Connection Pooling:**
```python
# Each request creates new connection (expensive!)
for i in range(100):
    conn = psycopg2.connect(...)  # Expensive: TCP handshake, authentication
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (i,))
    conn.close()

# Total time: ~5 seconds (connection overhead)
```

**With Connection Pooling:**
```python
from psycopg2 import pool

# Create connection pool once
connection_pool = pool.SimpleConnectionPool(
    minconn=5,
    maxconn=20,
    host='localhost',
    database='mydb'
)

# Reuse connections
for i in range(100):
    conn = connection_pool.getconn()  # Fast: Reuses existing connection
    cursor = conn.cursor()
    cursor.execute("SELECT * FROM users WHERE id = %s", (i,))
    connection_pool.putconn(conn)

# Total time: ~0.5 seconds ⚡ 10x faster!
```

#### 6. Operating System Page Cache

OS caches file system pages in RAM automatically.

**How it works:**
- Database stores data in files on disk
- OS caches frequently accessed file pages
- Reduces physical disk I/O

**Check OS cache usage:**
```bash
# Linux
free -h
# Look at "cached" memory

# Monitor cache effectiveness
vmstat 1
# Watch "bi" (blocks in) and "bo" (blocks out)
```

#### 7. CDN / Edge Caching (for read-heavy APIs)

Cache database query results at edge locations.

**Example Architecture:**
```
User Request → CDN (cached response) → API Server → Redis → Database
                ↑ 200ms                  ↑ 50ms      ↑ 10ms    ↑ 100ms
```

---

**Caching Strategies:**

**1. Cache-Aside (Lazy Loading)**
```python
def get_product(product_id):
    # Check cache
    cached = redis.get(f"product:{product_id}")
    if cached:
        return cached
    
    # Query database
    product = db.query("SELECT * FROM products WHERE id = ?", product_id)
    
    # Store in cache
    redis.set(f"product:{product_id}", product, ex=3600)
    
    return product
```

**2. Write-Through Cache**
```python
def update_product(product_id, data):
    # Update database
    db.execute("UPDATE products SET data = ? WHERE id = ?", data, product_id)
    
    # Update cache immediately
    redis.set(f"product:{product_id}", data, ex=3600)
```

**3. Write-Behind Cache (Write-Back)**
```python
def update_product(product_id, data):
    # Update cache immediately
    redis.set(f"product:{product_id}", data, ex=3600)
    
    # Queue database write for later (async)
    queue.enqueue(update_database, product_id, data)
```

---

**Cache Invalidation (The Hard Part):**

**Problem:** Keeping cache and database in sync.

**Strategies:**

**1. TTL (Time To Live)**
```python
# Cache expires after 1 hour
redis.setex("user:12345", 3600, user_data)
```

**2. Manual Invalidation**
```python
def update_user(user_id, data):
    db.execute("UPDATE users SET data = ? WHERE id = ?", data, user_id)
    redis.delete(f"user:{user_id}")  # Invalidate cache
```

**3. Event-Based Invalidation**
```python
# PostgreSQL NOTIFY/LISTEN
# When data changes, notify cache to invalidate
cursor.execute("NOTIFY cache_invalidate, 'user:12345'")
```

---

**Performance Improvement Examples:**

| Scenario | Without Cache | With Cache | Improvement |
|----------|--------------|------------|-------------|
| Simple query | 50ms | 0.5ms | 100x faster |
| Complex aggregation | 5000ms | 10ms | 500x faster |
| Connection creation | 100ms | 1ms | 100x faster |
| API response | 200ms | 5ms | 40x faster |

---

**Best Practices:**

1. ✅ Cache frequently accessed data
2. ✅ Cache expensive computations
3. ✅ Use appropriate TTL values
4. ✅ Monitor cache hit ratios
5. ✅ Implement cache invalidation strategy
6. ✅ Use connection pooling
7. ✅ Configure database buffer pool appropriately
8. ❌ Don't cache rapidly changing data
9. ❌ Don't cache user-specific sensitive data globally

---

**Monitoring Cache Performance:**

**Redis:**
```bash
redis-cli INFO stats
# Look at: keyspace_hits, keyspace_misses

# Calculate hit ratio
Hit Ratio = hits / (hits + misses) * 100
# Goal: > 80%
```

**PostgreSQL:**
```sql
-- Buffer cache hit ratio
SELECT 
  sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 as hit_ratio
FROM pg_statio_user_tables;
-- Goal: > 99%
```

**Application Metrics:**
```python
cache_hits = 0
cache_misses = 0

def get_cached_data(key):
    global cache_hits, cache_misses
    
    data = redis.get(key)
    if data:
        cache_hits += 1
        return data
    
    cache_misses += 1
    # Query database...
    
# Log metrics
print(f"Cache Hit Ratio: {cache_hits / (cache_hits + cache_misses) * 100}%")
```

