# II. SQL Databases (PostgreSQL Focused)

---

## A. SQL Querying (Practical)

### 1. Write a query to find top 3 products in each category.

This is a classic window function problem that requires ranking products within each category.

**Solution using ROW_NUMBER():**

```sql
WITH RankedProducts AS (
    SELECT 
        product_id,
        product_name,
        category_id,
        category_name,
        price,
        sales_count,
        ROW_NUMBER() OVER (
            PARTITION BY category_id 
            ORDER BY sales_count DESC
        ) as rank
    FROM products
    JOIN categories ON products.category_id = categories.id
)
SELECT 
    product_id,
    product_name,
    category_name,
    price,
    sales_count
FROM RankedProducts
WHERE rank <= 3
ORDER BY category_name, rank;
```

**Alternative using RANK() (handles ties differently):**

```sql
SELECT 
    product_id,
    product_name,
    category_name,
    price,
    sales_count
FROM (
    SELECT 
        p.product_id,
        p.product_name,
        c.category_name,
        p.price,
        p.sales_count,
        RANK() OVER (
            PARTITION BY p.category_id 
            ORDER BY p.sales_count DESC, p.price ASC
        ) as product_rank
    FROM products p
    JOIN categories c ON p.category_id = c.id
) ranked
WHERE product_rank <= 3
ORDER BY category_name, product_rank;
```

**Example Data and Output:**

```sql
-- Sample data
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    category_id INT,
    price DECIMAL(10,2),
    sales_count INT
);

INSERT INTO products VALUES
(1, 'Laptop Pro', 1, 1299.99, 500),
(2, 'Laptop Air', 1, 999.99, 800),
(3, 'Laptop Basic', 1, 599.99, 1200),
(4, 'Desktop Gaming', 1, 1599.99, 300),
(5, 'iPhone 15', 2, 999.99, 2000),
(6, 'Samsung Galaxy', 2, 899.99, 1500),
(7, 'Pixel 8', 2, 799.99, 800),
(8, 'OnePlus 12', 2, 699.99, 600);

-- Result:
| product_name    | category_name | sales_count | rank |
|----------------|---------------|-------------|------|
| Laptop Basic   | Computers     | 1200        | 1    |
| Laptop Air     | Computers     | 800         | 2    |
| Laptop Pro     | Computers     | 500         | 3    |
| iPhone 15      | Phones        | 2000        | 1    |
| Samsung Galaxy | Phones        | 1500        | 2    |
| Pixel 8        | Phones        | 800         | 3    |
```

**Using DENSE_RANK() (for different tie handling):**

```sql
-- DENSE_RANK() doesn't skip ranks after ties
SELECT 
    product_name,
    category_name,
    sales_count,
    DENSE_RANK() OVER (
        PARTITION BY category_id 
        ORDER BY sales_count DESC
    ) as dense_rank
FROM products
JOIN categories ON products.category_id = categories.id
WHERE dense_rank <= 3;
```

**Key Differences:**
- **ROW_NUMBER()**: Always unique, arbitrary ordering for ties (1, 2, 3, 4, 5)
- **RANK()**: Same rank for ties, skips next rank (1, 2, 2, 4, 5)
- **DENSE_RANK()**: Same rank for ties, no skipping (1, 2, 2, 3, 4)

---

### 2. How do you handle duplicates in SQL?

There are several strategies to handle duplicates depending on your use case.

**1. Finding Duplicates:**

```sql
-- Find duplicate rows based on specific columns
SELECT 
    email,
    COUNT(*) as duplicate_count
FROM users
GROUP BY email
HAVING COUNT(*) > 1;

-- Find all duplicate records with details
SELECT 
    u.*
FROM users u
INNER JOIN (
    SELECT email, COUNT(*) as cnt
    FROM users
    GROUP BY email
    HAVING COUNT(*) > 1
) duplicates ON u.email = duplicates.email
ORDER BY u.email;
```

**2. Removing Duplicates with DISTINCT:**

```sql
-- Get unique rows
SELECT DISTINCT email, name
FROM users;

-- DISTINCT on specific columns (PostgreSQL)
SELECT DISTINCT ON (email) 
    user_id,
    email,
    name,
    created_at
FROM users
ORDER BY email, created_at DESC;  -- Keeps most recent per email
```

**3. Deleting Duplicates - Keep First Occurrence:**

```sql
-- Using ROW_NUMBER() to identify duplicates
WITH RankedUsers AS (
    SELECT 
        user_id,
        email,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY created_at ASC  -- Keep oldest
        ) as rn
    FROM users
)
DELETE FROM users
WHERE user_id IN (
    SELECT user_id 
    FROM RankedUsers 
    WHERE rn > 1
);
```

**4. Deleting Duplicates - Keep Latest:**

```sql
-- Delete all but the most recent record
DELETE FROM users
WHERE user_id NOT IN (
    SELECT MAX(user_id)
    FROM users
    GROUP BY email
);

-- Alternative using CTE (PostgreSQL)
WITH LatestUsers AS (
    SELECT 
        user_id,
        ROW_NUMBER() OVER (
            PARTITION BY email 
            ORDER BY created_at DESC
        ) as rn
    FROM users
)
DELETE FROM users
WHERE user_id IN (
    SELECT user_id 
    FROM LatestUsers 
    WHERE rn > 1
);
```

**5. Using Temporary Table:**

```sql
-- Create temp table with distinct records
CREATE TEMP TABLE users_temp AS
SELECT DISTINCT ON (email)
    user_id,
    email,
    name,
    created_at
FROM users
ORDER BY email, created_at DESC;

-- Drop original table
DROP TABLE users;

-- Rename temp table
ALTER TABLE users_temp RENAME TO users;
```

**6. Preventing Duplicates:**

```sql
-- Add UNIQUE constraint
ALTER TABLE users ADD CONSTRAINT unique_email UNIQUE (email);

-- Create unique index
CREATE UNIQUE INDEX idx_users_email ON users(email);

-- Composite unique constraint
ALTER TABLE enrollments 
ADD CONSTRAINT unique_student_course 
UNIQUE (student_id, course_id);
```

**7. Handling Duplicates on INSERT:**

```sql
-- PostgreSQL: ON CONFLICT
INSERT INTO users (email, name)
VALUES ('john@example.com', 'John Doe')
ON CONFLICT (email) 
DO NOTHING;

-- Update on conflict
INSERT INTO users (email, name, login_count)
VALUES ('john@example.com', 'John Doe', 1)
ON CONFLICT (email) 
DO UPDATE SET 
    login_count = users.login_count + 1,
    last_login = NOW();

-- MySQL: INSERT IGNORE
INSERT IGNORE INTO users (email, name)
VALUES ('john@example.com', 'John Doe');

-- MySQL: ON DUPLICATE KEY UPDATE
INSERT INTO users (email, name, login_count)
VALUES ('john@example.com', 'John Doe', 1)
ON DUPLICATE KEY UPDATE 
    login_count = login_count + 1;
```

**8. Finding Duplicates Across Multiple Columns:**

```sql
-- Find duplicates based on multiple columns
SELECT 
    first_name,
    last_name,
    birth_date,
    COUNT(*) as count
FROM users
GROUP BY first_name, last_name, birth_date
HAVING COUNT(*) > 1;

-- Get all details of duplicates
SELECT u.*
FROM users u
INNER JOIN (
    SELECT first_name, last_name, birth_date
    FROM users
    GROUP BY first_name, last_name, birth_date
    HAVING COUNT(*) > 1
) dups 
ON u.first_name = dups.first_name 
   AND u.last_name = dups.last_name 
   AND u.birth_date = dups.birth_date
ORDER BY u.first_name, u.last_name, u.birth_date;
```

---

### 3. Difference between `INNER JOIN`, `LEFT JOIN`, and `FULL JOIN`.

JOINs combine rows from two or more tables based on a related column.

**Visual Representation:**

```
Table A (Customers)          Table B (Orders)
customer_id | name           order_id | customer_id | amount
-----------+-------          ---------+-------------+--------
1          | Alice          101      | 1           | 100
2          | Bob            102      | 1           | 200
3          | Charlie        103      | 3           | 150
4          | David          104      | 5           | 300
```

**1. INNER JOIN:**

Returns only matching rows from both tables.

```sql
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id;

-- Result: Only customers who have orders
| customer_id | name    | order_id | amount |
|-------------|---------|----------|--------|
| 1           | Alice   | 101      | 100    |
| 1           | Alice   | 102      | 200    |
| 3           | Charlie | 103      | 150    |

-- Alice (2 orders), Charlie (1 order)
-- Bob and David excluded (no orders)
```

**2. LEFT JOIN (LEFT OUTER JOIN):**

Returns all rows from the left table, with matching rows from the right table. NULL for non-matching rows.

```sql
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id;

-- Result: All customers, with order info if available
| customer_id | name    | order_id | amount |
|-------------|---------|----------|--------|
| 1           | Alice   | 101      | 100    |
| 1           | Alice   | 102      | 200    |
| 2           | Bob     | NULL     | NULL   |
| 3           | Charlie | 103      | 150    |
| 4           | David   | NULL     | NULL   |

-- All customers shown, even those without orders
```

**3. RIGHT JOIN (RIGHT OUTER JOIN):**

Returns all rows from the right table, with matching rows from the left table.

```sql
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
RIGHT JOIN orders o ON c.customer_id = o.customer_id;

-- Result: All orders, with customer info if available
| customer_id | name    | order_id | amount |
|-------------|---------|----------|--------|
| 1           | Alice   | 101      | 100    |
| 1           | Alice   | 102      | 200    |
| 3           | Charlie | 103      | 150    |
| NULL        | NULL    | 104      | 300    |

-- Order 104 shown even though customer_id 5 doesn't exist
```

**4. FULL OUTER JOIN:**

Returns all rows from both tables, with NULL where there's no match.

```sql
SELECT 
    c.customer_id,
    c.name,
    o.order_id,
    o.amount
FROM customers c
FULL OUTER JOIN orders o ON c.customer_id = o.customer_id;

-- Result: All customers AND all orders
| customer_id | name    | order_id | amount |
|-------------|---------|----------|--------|
| 1           | Alice   | 101      | 100    |
| 1           | Alice   | 102      | 200    |
| 2           | Bob     | NULL     | NULL   |
| 3           | Charlie | 103      | 150    |
| 4           | David   | NULL     | NULL   |
| NULL        | NULL    | 104      | 300    |

-- Shows customers without orders AND orders without valid customers
```

**5. CROSS JOIN:**

Cartesian product - every row from table A combined with every row from table B.

```sql
SELECT 
    c.name,
    p.product_name
FROM customers c
CROSS JOIN products p;

-- If 4 customers and 3 products = 12 rows (4 × 3)
```

**Practical Examples:**

**Find customers without orders:**
```sql
SELECT c.*
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
WHERE o.order_id IS NULL;
```

**Find orders without valid customers (orphaned records):**
```sql
SELECT o.*
FROM orders o
LEFT JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id IS NULL;
```

**Multiple JOINs:**
```sql
SELECT 
    c.name as customer_name,
    o.order_id,
    o.order_date,
    p.product_name,
    oi.quantity,
    oi.price
FROM customers c
INNER JOIN orders o ON c.customer_id = o.customer_id
INNER JOIN order_items oi ON o.order_id = oi.order_id
INNER JOIN products p ON oi.product_id = p.product_id
WHERE o.order_date >= '2024-01-01'
ORDER BY c.name, o.order_date;
```

**Self JOIN (joining table to itself):**
```sql
-- Find employees and their managers
SELECT 
    e.employee_name as employee,
    m.employee_name as manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.employee_id;
```

**Comparison Table:**

| JOIN Type | Returns | When to Use |
|-----------|---------|-------------|
| INNER JOIN | Only matching rows | When you need data that exists in both tables |
| LEFT JOIN | All left + matching right | When you need all records from main table |
| RIGHT JOIN | All right + matching left | Rarely used (can use LEFT JOIN instead) |
| FULL OUTER JOIN | All from both tables | When you need everything, matches or not |
| CROSS JOIN | Cartesian product | Combinations, testing, lookup tables |

---

### 4. How does `GROUP BY` and `HAVING` differ?

**GROUP BY** and **HAVING** work together but serve different purposes in aggregation queries.

**GROUP BY:**
- Groups rows with same values into summary rows
- Used with aggregate functions (COUNT, SUM, AVG, MAX, MIN)
- Filters rows BEFORE aggregation

**HAVING:**
- Filters groups AFTER aggregation
- Works with aggregate functions
- Applied after GROUP BY

**Key Difference: WHERE vs HAVING:**
- **WHERE** filters individual rows BEFORE grouping
- **HAVING** filters groups AFTER aggregation

**Example 1: Basic GROUP BY**

```sql
-- Count users by country
SELECT 
    country,
    COUNT(*) as user_count
FROM users
GROUP BY country;

-- Result:
| country | user_count |
|---------|-----------|
| USA     | 1500      |
| Canada  | 800       |
| UK      | 600       |
```

**Example 2: GROUP BY with HAVING**

```sql
-- Find countries with more than 1000 users
SELECT 
    country,
    COUNT(*) as user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 1000;

-- Result:
| country | user_count |
|---------|-----------|
| USA     | 1500      |

-- Canada and UK excluded because count < 1000
```

**Example 3: WHERE vs HAVING**

```sql
-- WHERE: Filter rows BEFORE grouping
SELECT 
    country,
    COUNT(*) as user_count
FROM users
WHERE registration_date >= '2024-01-01'  -- Filter individual rows first
GROUP BY country
HAVING COUNT(*) > 100;  -- Then filter groups

-- Execution order:
-- 1. WHERE filters rows (only 2024 registrations)
-- 2. GROUP BY groups remaining rows
-- 3. HAVING filters groups (only countries with > 100 users)
```

**Example 4: Multiple Aggregations with HAVING**

```sql
-- Find products with average rating > 4.5 and more than 100 reviews
SELECT 
    product_id,
    product_name,
    AVG(rating) as avg_rating,
    COUNT(*) as review_count
FROM reviews
GROUP BY product_id, product_name
HAVING AVG(rating) > 4.5 
   AND COUNT(*) > 100
ORDER BY avg_rating DESC;
```

**Example 5: Complex Example**

```sql
-- Find top-selling categories in 2024 with more than $10,000 in sales
SELECT 
    c.category_name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(oi.quantity * oi.price) as total_sales,
    AVG(oi.price) as avg_price
FROM categories c
JOIN products p ON c.category_id = p.category_id
JOIN order_items oi ON p.product_id = oi.product_id
JOIN orders o ON oi.order_id = o.order_id
WHERE o.order_date BETWEEN '2024-01-01' AND '2024-12-31'  -- Filter rows
  AND o.status = 'completed'
GROUP BY c.category_name
HAVING SUM(oi.quantity * oi.price) > 10000  -- Filter groups
ORDER BY total_sales DESC;
```

**Example 6: HAVING with Calculated Columns**

```sql
-- Find customers who spent more than $1000 on average per order
SELECT 
    customer_id,
    customer_name,
    COUNT(order_id) as order_count,
    SUM(total_amount) as total_spent,
    AVG(total_amount) as avg_order_value
FROM orders
JOIN customers USING (customer_id)
WHERE order_date >= '2024-01-01'
GROUP BY customer_id, customer_name
HAVING AVG(total_amount) > 1000
   AND COUNT(order_id) >= 5;  -- At least 5 orders
```

**Example 7: HAVING with String Aggregation (PostgreSQL)**

```sql
-- Find users with multiple email addresses
SELECT 
    user_id,
    first_name,
    last_name,
    STRING_AGG(email, ', ') as emails,
    COUNT(*) as email_count
FROM user_emails
GROUP BY user_id, first_name, last_name
HAVING COUNT(*) > 1;
```

**Common Mistakes:**

```sql
-- ❌ WRONG: Using WHERE with aggregate functions
SELECT country, COUNT(*) as user_count
FROM users
WHERE COUNT(*) > 1000  -- ERROR! Use HAVING instead
GROUP BY country;

-- ✅ CORRECT:
SELECT country, COUNT(*) as user_count
FROM users
GROUP BY country
HAVING COUNT(*) > 1000;

-- ❌ WRONG: Using non-aggregated column not in GROUP BY
SELECT country, city, COUNT(*) 
FROM users
GROUP BY country;  -- ERROR! city must be in GROUP BY

-- ✅ CORRECT:
SELECT country, city, COUNT(*) 
FROM users
GROUP BY country, city;
```

**Query Execution Order:**

```sql
-- SQL execution order (conceptual):
FROM users                           -- 1. Get data
WHERE registration_date >= '2024'    -- 2. Filter rows
GROUP BY country                     -- 3. Group rows
HAVING COUNT(*) > 1000              -- 4. Filter groups
SELECT country, COUNT(*)             -- 5. Select columns
ORDER BY COUNT(*) DESC               -- 6. Sort results
LIMIT 10;                           -- 7. Limit output
```

**Summary Table:**

| Aspect | WHERE | HAVING |
|--------|-------|--------|
| Applied | Before GROUP BY | After GROUP BY |
| Filters | Individual rows | Groups |
| Can use aggregate functions? | No | Yes |
| Example | `WHERE age > 18` | `HAVING COUNT(*) > 10` |
| Performance | Faster (filters early) | Slower (filters after grouping) |

---

### 5. What are window functions? Give examples using `ROW_NUMBER`, `RANK`, and `LAG`.

**Window Functions** (also called analytic functions) perform calculations across a set of rows related to the current row, without collapsing the result set like GROUP BY does.

**Key Characteristics:**
- Don't reduce number of rows (unlike GROUP BY)
- Can access other rows in the "window" of data
- Powerful for rankings, running totals, moving averages, etc.

**Syntax:**
```sql
function_name() OVER (
    [PARTITION BY column]
    [ORDER BY column]
    [ROWS or RANGE clause]
)
```

---

### ROW_NUMBER()

Assigns a unique sequential number to each row within a partition.

**Example 1: Basic ROW_NUMBER()**

```sql
SELECT 
    employee_id,
    employee_name,
    department,
    salary,
    ROW_NUMBER() OVER (ORDER BY salary DESC) as salary_rank
FROM employees;

-- Result:
| employee_name | department | salary  | salary_rank |
|--------------|------------|---------|-------------|
| Alice        | Sales      | 150000  | 1           |
| Bob          | IT         | 145000  | 2           |
| Charlie      | Sales      | 140000  | 3           |
| David        | IT         | 135000  | 4           |
```

**Example 2: ROW_NUMBER() with PARTITION BY**

```sql
-- Number employees within each department
SELECT 
    employee_name,
    department,
    salary,
    ROW_NUMBER() OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as dept_salary_rank
FROM employees;

-- Result:
| employee_name | department | salary  | dept_salary_rank |
|--------------|------------|---------|-----------------|
| Alice        | Sales      | 150000  | 1               |
| Charlie      | Sales      | 140000  | 2               |
| Eve          | Sales      | 130000  | 3               |
| Bob          | IT         | 145000  | 1               |
| David        | IT         | 135000  | 2               |
```

**Example 3: Pagination with ROW_NUMBER()**

```sql
-- Get page 2 (rows 11-20) of products
WITH NumberedProducts AS (
    SELECT 
        product_id,
        product_name,
        price,
        ROW_NUMBER() OVER (ORDER BY product_name) as row_num
    FROM products
)
SELECT product_id, product_name, price
FROM NumberedProducts
WHERE row_num BETWEEN 11 AND 20;
```

---

### RANK() and DENSE_RANK()

**RANK()**: Assigns same rank to ties, skips next rank  
**DENSE_RANK()**: Assigns same rank to ties, no gap in ranks

**Example 4: Comparing RANK, DENSE_RANK, ROW_NUMBER**

```sql
SELECT 
    student_name,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) as row_num,
    RANK() OVER (ORDER BY score DESC) as rank,
    DENSE_RANK() OVER (ORDER BY score DESC) as dense_rank
FROM exam_scores;

-- Result:
| student_name | score | row_num | rank | dense_rank |
|-------------|-------|---------|------|-----------|
| Alice       | 95    | 1       | 1    | 1         |
| Bob         | 92    | 2       | 2    | 2         |
| Charlie     | 92    | 3       | 2    | 2         |  ← Same rank
| David       | 88    | 4       | 4    | 3         |  ← RANK skips 3
| Eve         | 85    | 5       | 5    | 4         |
```

**Example 5: Top N per Category**

```sql
-- Get top 3 highest-paid employees per department
WITH RankedEmployees AS (
    SELECT 
        employee_name,
        department,
        salary,
        RANK() OVER (
            PARTITION BY department 
            ORDER BY salary DESC
        ) as salary_rank
    FROM employees
)
SELECT *
FROM RankedEmployees
WHERE salary_rank <= 3;
```

---

### LAG() and LEAD()

**LAG()**: Access previous row's value  
**LEAD()**: Access next row's value

**Example 6: LAG() - Compare with Previous Value**

```sql
-- Calculate day-over-day change in stock price
SELECT 
    date,
    stock_price,
    LAG(stock_price) OVER (ORDER BY date) as prev_day_price,
    stock_price - LAG(stock_price) OVER (ORDER BY date) as daily_change,
    ROUND(
        (stock_price - LAG(stock_price) OVER (ORDER BY date)) 
        / LAG(stock_price) OVER (ORDER BY date) * 100, 
        2
    ) as pct_change
FROM stock_prices
ORDER BY date;

-- Result:
| date       | stock_price | prev_day_price | daily_change | pct_change |
|-----------|-------------|----------------|--------------|-----------|
| 2024-01-01| 100.00      | NULL           | NULL         | NULL      |
| 2024-01-02| 102.50      | 100.00         | 2.50         | 2.50      |
| 2024-01-03| 101.00      | 102.50         | -1.50        | -1.46     |
```

**Example 7: LEAD() - Look Ahead**

```sql
-- Find users who registered and then made a purchase
SELECT 
    user_id,
    action_type,
    action_date,
    LEAD(action_type) OVER (
        PARTITION BY user_id 
        ORDER BY action_date
    ) as next_action,
    LEAD(action_date) OVER (
        PARTITION BY user_id 
        ORDER BY action_date
    ) as next_action_date
FROM user_actions
WHERE action_type IN ('registration', 'purchase');
```

**Example 8: Calculate Gaps Between Events**

```sql
-- Time between consecutive orders for each customer
SELECT 
    customer_id,
    order_id,
    order_date,
    LAG(order_date) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as prev_order_date,
    order_date - LAG(order_date) OVER (
        PARTITION BY customer_id 
        ORDER BY order_date
    ) as days_since_last_order
FROM orders
ORDER BY customer_id, order_date;
```

---

### Other Common Window Functions

**FIRST_VALUE() and LAST_VALUE()**

```sql
-- Compare each employee's salary to highest and lowest in department
SELECT 
    employee_name,
    department,
    salary,
    FIRST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
    ) as dept_highest_salary,
    LAST_VALUE(salary) OVER (
        PARTITION BY department 
        ORDER BY salary DESC
        ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
    ) as dept_lowest_salary
FROM employees;
```

**Running Totals with SUM()**

```sql
-- Calculate running total of sales
SELECT 
    order_date,
    daily_sales,
    SUM(daily_sales) OVER (
        ORDER BY order_date
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) as running_total,
    AVG(daily_sales) OVER (
        ORDER BY order_date
        ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
    ) as moving_avg_7days
FROM daily_sales
ORDER BY order_date;
```

**NTILE() - Divide into N Groups**

```sql
-- Divide customers into 4 quartiles based on spending
SELECT 
    customer_id,
    customer_name,
    total_spent,
    NTILE(4) OVER (ORDER BY total_spent DESC) as spending_quartile
FROM customer_totals;

-- Quartile 1 = top 25% spenders
-- Quartile 4 = bottom 25% spenders
```

**Percent Rank**

```sql
-- Calculate percentile ranking
SELECT 
    student_name,
    score,
    PERCENT_RANK() OVER (ORDER BY score) as percentile,
    CUME_DIST() OVER (ORDER BY score) as cumulative_distribution
FROM exam_scores;
```

---

**Complete Real-World Example: Sales Analysis**

```sql
WITH SalesAnalysis AS (
    SELECT 
        sales_date,
        salesperson_name,
        sales_amount,
        
        -- Ranking
        ROW_NUMBER() OVER (
            PARTITION BY sales_date 
            ORDER BY sales_amount DESC
        ) as daily_rank,
        
        RANK() OVER (
            ORDER BY sales_amount DESC
        ) as overall_rank,
        
        -- Comparison to previous
        LAG(sales_amount) OVER (
            PARTITION BY salesperson_name 
            ORDER BY sales_date
        ) as prev_day_sales,
        
        -- Running totals
        SUM(sales_amount) OVER (
            PARTITION BY salesperson_name 
            ORDER BY sales_date
            ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
        ) as running_total,
        
        -- Moving average
        AVG(sales_amount) OVER (
            PARTITION BY salesperson_name 
            ORDER BY sales_date
            ROWS BETWEEN 6 PRECEDING AND CURRENT ROW
        ) as moving_avg_7days,
        
        -- Percentile
        PERCENT_RANK() OVER (
            PARTITION BY sales_date 
            ORDER BY sales_amount
        ) as daily_percentile
        
    FROM sales
)
SELECT 
    sales_date,
    salesperson_name,
    sales_amount,
    daily_rank,
    prev_day_sales,
    sales_amount - prev_day_sales as day_over_day_change,
    running_total,
    ROUND(moving_avg_7days, 2) as avg_7d,
    ROUND(daily_percentile * 100, 2) as percentile
FROM SalesAnalysis
ORDER BY sales_date DESC, daily_rank;
```

**Summary:**

| Function | Purpose | Example Use Case |
|----------|---------|------------------|
| ROW_NUMBER() | Unique sequential number | Pagination, deduplication |
| RANK() | Ranking with gaps for ties | Competition rankings |
| DENSE_RANK() | Ranking without gaps | Grade rankings |
| LAG() | Access previous row | Day-over-day changes |
| LEAD() | Access next row | Future predictions |
| FIRST_VALUE() | First value in window | Compare to best |
| LAST_VALUE() | Last value in window | Compare to worst |
| SUM() OVER | Running total | Cumulative sales |
| AVG() OVER | Moving average | Trend analysis |
| NTILE() | Divide into groups | Quartile analysis |


## B. Advanced SQL Concepts

### 1. Explain CTE (Common Table Expressions) and recursive queries.

**Common Table Expression (CTE):**

A CTE is a temporary named result set that exists only during the execution of a query. It makes complex queries more readable and maintainable.

**Syntax:**
```sql
WITH cte_name AS (
    -- Query definition
    SELECT ...
)
SELECT * FROM cte_name;
```

**Basic CTE Example:**

```sql
-- Without CTE (nested subquery - hard to read)
SELECT *
FROM (
    SELECT customer_id, SUM(total_amount) as total_spent
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY customer_id
) customer_totals
WHERE total_spent > 1000;

-- With CTE (much cleaner!)
WITH customer_totals AS (
    SELECT 
        customer_id,
        SUM(total_amount) as total_spent
    FROM orders
    WHERE order_date >= '2024-01-01'
    GROUP BY customer_id
)
SELECT *
FROM customer_totals
WHERE total_spent > 1000;
```

**Multiple CTEs:**

```sql
-- Chain multiple CTEs together
WITH 
active_customers AS (
    SELECT customer_id, customer_name
    FROM customers
    WHERE is_active = true
),
customer_orders AS (
    SELECT 
        ac.customer_id,
        ac.customer_name,
        COUNT(o.order_id) as order_count,
        SUM(o.total_amount) as total_spent
    FROM active_customers ac
    LEFT JOIN orders o ON ac.customer_id = o.customer_id
    GROUP BY ac.customer_id, ac.customer_name
),
top_customers AS (
    SELECT *
    FROM customer_orders
    WHERE order_count >= 5
    ORDER BY total_spent DESC
    LIMIT 10
)
SELECT * FROM top_customers;
```

**CTE Benefits:**
- Improved readability
- Can be referenced multiple times in main query
- Makes debugging easier
- Can be recursive (see below)

---

**Recursive CTEs:**

Recursive CTEs reference themselves to solve hierarchical or iterative problems.

**Syntax:**
```sql
WITH RECURSIVE cte_name AS (
    -- Base case (anchor member)
    SELECT ...
    
    UNION ALL
    
    -- Recursive case (recursive member)
    SELECT ...
    FROM cte_name
    WHERE termination_condition
)
SELECT * FROM cte_name;
```

**Example 1: Generate Number Sequence**

```sql
-- Generate numbers 1 to 10
WITH RECURSIVE numbers AS (
    -- Base case
    SELECT 1 as n
    
    UNION ALL
    
    -- Recursive case
    SELECT n + 1
    FROM numbers
    WHERE n < 10
)
SELECT * FROM numbers;

-- Result: 1, 2, 3, 4, 5, 6, 7, 8, 9, 10
```

**Example 2: Organizational Hierarchy (Employee-Manager)**

```sql
CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    manager_id INT
);

INSERT INTO employees VALUES
(1, 'CEO Alice', NULL),
(2, 'VP Bob', 1),
(3, 'VP Charlie', 1),
(4, 'Manager David', 2),
(5, 'Manager Eve', 2),
(6, 'Employee Frank', 4),
(7, 'Employee Grace', 4),
(8, 'Employee Henry', 5);

-- Find all employees under Bob (including indirect reports)
WITH RECURSIVE employee_hierarchy AS (
    -- Base case: Start with Bob
    SELECT 
        employee_id,
        employee_name,
        manager_id,
        1 as level,
        employee_name as path
    FROM employees
    WHERE employee_name = 'VP Bob'
    
    UNION ALL
    
    -- Recursive case: Find employees reporting to current level
    SELECT 
        e.employee_id,
        e.employee_name,
        e.manager_id,
        eh.level + 1,
        eh.path || ' -> ' || e.employee_name
    FROM employees e
    INNER JOIN employee_hierarchy eh ON e.manager_id = eh.employee_id
)
SELECT * FROM employee_hierarchy
ORDER BY level, employee_name;

-- Result:
| employee_id | employee_name    | level | path                              |
|------------|------------------|-------|-----------------------------------|
| 2          | VP Bob           | 1     | VP Bob                            |
| 4          | Manager David    | 2     | VP Bob -> Manager David           |
| 5          | Manager Eve      | 2     | VP Bob -> Manager Eve             |
| 6          | Employee Frank   | 3     | VP Bob -> Manager David -> Frank  |
| 7          | Employee Grace   | 3     | VP Bob -> Manager David -> Grace  |
| 8          | Employee Henry   | 3     | VP Bob -> Manager Eve -> Henry    |
```

**Example 3: Category Tree (Parent-Child Relationships)**

```sql
CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100),
    parent_id INT
);

-- Electronics > Computers > Laptops
-- Electronics > Phones > Smartphones

WITH RECURSIVE category_tree AS (
    -- Base case: Top-level categories
    SELECT 
        category_id,
        category_name,
        parent_id,
        0 as depth,
        category_name as full_path
    FROM categories
    WHERE parent_id IS NULL
    
    UNION ALL
    
    -- Recursive case: Child categories
    SELECT 
        c.category_id,
        c.category_name,
        c.parent_id,
        ct.depth + 1,
        ct.full_path || ' > ' || c.category_name
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.category_id
)
SELECT 
    REPEAT('  ', depth) || category_name as indented_category,
    depth,
    full_path
FROM category_tree
ORDER BY full_path;
```

**Example 4: Calculate Factorial**

```sql
-- Calculate 5! = 5 × 4 × 3 × 2 × 1 = 120
WITH RECURSIVE factorial AS (
    SELECT 
        5 as n,
        1 as result
    
    UNION ALL
    
    SELECT 
        n - 1,
        result * n
    FROM factorial
    WHERE n > 1
)
SELECT result as factorial_5
FROM factorial
WHERE n = 1;

-- Result: 120
```

**Example 5: Find All Direct and Indirect Dependencies**

```sql
CREATE TABLE tasks (
    task_id INT PRIMARY KEY,
    task_name VARCHAR(100),
    depends_on_task_id INT
);

-- Find all dependencies of a given task (transitive dependencies)
WITH RECURSIVE task_dependencies AS (
    -- Base case: Direct dependencies
    SELECT 
        task_id,
        task_name,
        depends_on_task_id,
        1 as dependency_level
    FROM tasks
    WHERE task_id = 10  -- Starting task
    
    UNION ALL
    
    -- Recursive case: Dependencies of dependencies
    SELECT 
        t.task_id,
        t.task_name,
        t.depends_on_task_id,
        td.dependency_level + 1
    FROM tasks t
    INNER JOIN task_dependencies td ON t.task_id = td.depends_on_task_id
)
SELECT DISTINCT
    task_name,
    dependency_level
FROM task_dependencies
WHERE task_id != 10
ORDER BY dependency_level;
```

**Example 6: Graph Traversal (Find all connected nodes)**

```sql
-- Social network: Find all friends and friends-of-friends
WITH RECURSIVE friend_network AS (
    -- Base case: Direct friends
    SELECT 
        user_id,
        friend_id,
        1 as degree
    FROM friendships
    WHERE user_id = 100
    
    UNION
    
    -- Recursive case: Friends of friends
    SELECT 
        fn.user_id,
        f.friend_id,
        fn.degree + 1
    FROM friend_network fn
    INNER JOIN friendships f ON fn.friend_id = f.user_id
    WHERE fn.degree < 3  -- Limit to 3 degrees of separation
)
SELECT DISTINCT 
    friend_id,
    MIN(degree) as closest_degree
FROM friend_network
GROUP BY friend_id
ORDER BY closest_degree, friend_id;
```

**Example 7: Date Range Generation**

```sql
-- Generate all dates in a month
WITH RECURSIVE date_series AS (
    SELECT DATE '2024-01-01' as date
    
    UNION ALL
    
    SELECT date + INTERVAL '1 day'
    FROM date_series
    WHERE date < DATE '2024-01-31'
)
SELECT 
    date,
    TO_CHAR(date, 'Day') as day_name
FROM date_series;
```

**Important Considerations:**

1. **Infinite Loop Protection**: Always have a termination condition
```sql
-- ❌ BAD: Infinite loop
WITH RECURSIVE bad_recursion AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM bad_recursion  -- No WHERE clause!
)
SELECT * FROM bad_recursion;

-- ✅ GOOD: Termination condition
WITH RECURSIVE good_recursion AS (
    SELECT 1 as n
    UNION ALL
    SELECT n + 1 FROM bad_recursion WHERE n < 1000  -- Stops at 1000
)
SELECT * FROM good_recursion;
```

2. **Cycle Detection**:
```sql
-- Detect and prevent cycles in hierarchical data
WITH RECURSIVE hierarchy AS (
    SELECT 
        id, 
        parent_id,
        ARRAY[id] as path,
        false as is_cycle
    FROM nodes
    WHERE parent_id IS NULL
    
    UNION ALL
    
    SELECT 
        n.id,
        n.parent_id,
        h.path || n.id,
        n.id = ANY(h.path) as is_cycle
    FROM nodes n
    INNER JOIN hierarchy h ON n.parent_id = h.id
    WHERE NOT h.is_cycle  -- Stop if cycle detected
)
SELECT * FROM hierarchy;
```

**CTE vs Subquery vs Temp Table:**

| Feature | CTE | Subquery | Temp Table |
|---------|-----|----------|------------|
| Scope | Single query | Single query | Session |
| Reusability | Can reference multiple times | Once per use | Multiple queries |
| Performance | Similar to subquery | Can be slower | Can be faster (indexed) |
| Readability | High | Low | Medium |
| Recursive | Yes | No | No |

---

### 2. How does indexing work in PostgreSQL (B-tree, GIN, GiST)?

PostgreSQL supports multiple index types, each optimized for different use cases.

---

#### **1. B-Tree Index (Default)**

**What it is:**
- Balanced tree structure
- Default index type in PostgreSQL
- Good for equality and range queries
- Keeps data sorted

**How it works:**
- Organizes data in a tree structure
- Each node contains keys and pointers
- Root → Branch nodes → Leaf nodes (actual data)
- Leaf nodes form a doubly-linked list for range scans

**Use Cases:**
- Numeric comparisons (`=`, `<`, `>`, `<=`, `>=`, `BETWEEN`)
- String comparisons with `LIKE 'prefix%'` (not `'%suffix'`)
- Sorting (`ORDER BY`)
- Primary keys and foreign keys

**Examples:**

```sql
-- Create B-tree index (default type)
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_products_price ON products(price);

-- Queries that benefit
SELECT * FROM users WHERE email = 'user@example.com';  -- Equality
SELECT * FROM orders WHERE order_date >= '2024-01-01';  -- Range
SELECT * FROM products WHERE price BETWEEN 100 AND 500;  -- Range
SELECT * FROM users ORDER BY email;  -- Sorting

-- Multi-column B-tree index
CREATE INDEX idx_orders_customer_date ON orders(customer_id, order_date);

-- Benefits queries:
SELECT * FROM orders 
WHERE customer_id = 123 AND order_date >= '2024-01-01';
```

**B-tree Structure:**
```
                [50]                    ← Root
               /    \
          [25]        [75]              ← Branch nodes
         /   \       /    \
     [10,20][30,40][60,70][80,90]      ← Leaf nodes (sorted)
```

**Performance:**
- Lookup: O(log n)
- Insert: O(log n)
- Delete: O(log n)
- Range scan: O(log n + k) where k = number of rows returned

---

#### **2. GIN (Generalized Inverted Index)**

**What it is:**
- Inverted index for composite values
- Stores mapping from values to rows containing those values
- Optimized for values that contain multiple component values

**How it works:**
- Breaks down composite values into individual elements
- Creates inverted index: element → list of rows
- Good for "contains" queries

**Use Cases:**
- Full-text search
- JSONB queries
- Array queries
- `tsvector` (text search)

**Examples:**

**Full-Text Search:**
```sql
-- Create GIN index for full-text search
CREATE INDEX idx_articles_content_gin 
ON articles USING GIN(to_tsvector('english', content));

-- Query that benefits
SELECT * FROM articles
WHERE to_tsvector('english', content) @@ to_tsquery('postgresql & indexing');

-- Much faster than:
SELECT * FROM articles WHERE content LIKE '%postgresql%' AND content LIKE '%indexing%';
```

**JSONB Index:**
```sql
-- Sample JSONB data
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    attributes JSONB
);

-- attributes might be: {"color": "red", "size": "large", "tags": ["sale", "popular"]}

-- Create GIN index on JSONB column
CREATE INDEX idx_products_attributes ON products USING GIN(attributes);

-- Queries that benefit
SELECT * FROM products 
WHERE attributes @> '{"color": "red"}';  -- Contains

SELECT * FROM products 
WHERE attributes ? 'size';  -- Key exists

SELECT * FROM products 
WHERE attributes @> '{"tags": ["sale"]}';  -- Array contains
```

**Array Index:**
```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    tags TEXT[]
);

-- Create GIN index on array
CREATE INDEX idx_posts_tags ON posts USING GIN(tags);

-- Queries that benefit
SELECT * FROM posts WHERE tags @> ARRAY['postgresql'];  -- Contains element
SELECT * FROM posts WHERE tags && ARRAY['database', 'sql'];  -- Overlaps
```

**GIN Structure (Conceptual):**
```
Inverted Index:
"postgresql" → [row1, row5, row12, row20]
"database"   → [row1, row3, row8, row15]
"indexing"   → [row5, row12, row18]

Query: "postgresql & indexing"
→ Intersect [row1, row5, row12, row20] and [row5, row12, row18]
→ Result: [row5, row12]
```

**GIN Performance:**
- Good for read-heavy workloads
- Slower writes (index maintenance)
- Larger index size
- Very fast for containment queries

---

#### **3. GiST (Generalized Search Tree)**

**What it is:**
- Balanced tree structure (like B-tree)
- Extensible framework supporting various data types
- Lossy index (may have false positives, requires recheck)

**How it works:**
- Hierarchical structure with bounding boxes
- Each node represents a region/range
- Supports custom operators and data types

**Use Cases:**
- Geometric data (PostGIS)
- Full-text search (less common than GIN)
- Range types
- Network addresses (inet)
- Custom data types

**Examples:**

**Geometric Queries (PostGIS):**
```sql
-- Create GiST index for spatial data
CREATE INDEX idx_locations_point ON locations USING GiST(coordinates);

-- Queries that benefit
SELECT * FROM locations 
WHERE ST_DWithin(coordinates, ST_MakePoint(-122.4, 37.8), 1000);  -- Within 1km

SELECT * FROM locations 
WHERE ST_Intersects(coordinates, some_polygon);
```

**Range Types:**
```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_id INT,
    date_range DATERANGE
);

-- Create GiST index on range type
CREATE INDEX idx_reservations_range ON reservations USING GiST(date_range);

-- Queries that benefit
SELECT * FROM reservations 
WHERE date_range && '[2024-01-15, 2024-01-20]'::daterange;  -- Overlaps

SELECT * FROM reservations 
WHERE date_range @> '2024-01-17'::date;  -- Contains date
```

**Full-Text Search (alternative to GIN):**
```sql
CREATE INDEX idx_documents_fts_gist 
ON documents USING GiST(to_tsvector('english', content));

-- Generally GIN is preferred for full-text search
-- GiST is faster to update but slower to query
```

**GiST Structure (Conceptual):**
```
For spatial data:
                [Bounding Box: entire region]
               /                             \
    [BB: North region]              [BB: South region]
    /              \                 /              \
[BB: NW]      [BB: NE]          [BB: SW]       [BB: SE]
  |  |          |  |              |  |           |  |
Points        Points            Points         Points
```

---

#### **4. BRIN (Block Range Index)** - Bonus

**What it is:**
- Stores summaries of values in consecutive physical blocks
- Very small index size
- Good for large tables with natural ordering

**Example:**
```sql
-- Perfect for time-series data or auto-increment IDs
CREATE INDEX idx_logs_timestamp_brin 
ON logs USING BRIN(timestamp);

-- Very small index, but efficient for:
SELECT * FROM logs WHERE timestamp >= '2024-01-01';
```

---

#### **5. Hash Index**

**What it is:**
- Hash table based index
- Only supports equality operations

**Example:**
```sql
CREATE INDEX idx_sessions_hash ON sessions USING HASH(session_id);

-- Only benefits equality:
SELECT * FROM sessions WHERE session_id = 'abc123';

-- Not useful for:
SELECT * FROM sessions WHERE session_id > 'abc123';  -- Won't use hash index
```

---

**Index Comparison Table:**

| Index Type | Best For | Pros | Cons | Example Use |
|------------|----------|------|------|-------------|
| **B-tree** | Equality, ranges, sorting | Fast, balanced | Standard overhead | `age > 18`, `ORDER BY name` |
| **GIN** | Composite values, full-text | Very fast containment | Slow writes, large size | JSONB, arrays, full-text |
| **GiST** | Geometric, ranges | Extensible, spatial | Lossy (rechecks) | PostGIS, IP ranges |
| **BRIN** | Sequential data | Tiny size | Only if naturally ordered | Time-series, logs |
| **Hash** | Equality only | Fast equality | No ranges | Session lookups |

---

**Choosing the Right Index:**

```sql
-- Equality and ranges → B-tree
CREATE INDEX idx_users_age ON users(age);

-- Full-text search → GIN
CREATE INDEX idx_articles_fts ON articles 
USING GIN(to_tsvector('english', content));

-- JSONB queries → GIN
CREATE INDEX idx_data_json ON data USING GIN(json_column);

-- Arrays → GIN
CREATE INDEX idx_tags ON posts USING GIN(tags);

-- Spatial data → GiST
CREATE INDEX idx_locations ON stores USING GiST(coordinates);

-- Range types → GiST
CREATE INDEX idx_availability ON bookings USING GiST(time_range);

-- Time-series (large, ordered) → BRIN
CREATE INDEX idx_events_time ON events USING BRIN(event_timestamp);
```

**Index Monitoring:**

```sql
-- Check if index is being used
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan as times_used,
    idx_tup_read as tuples_read,
    idx_tup_fetch as tuples_fetched
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY idx_scan DESC;

-- Find unused indexes
SELECT 
    schemaname || '.' || tablename as table,
    indexname as index,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Index size
SELECT 
    indexname,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE schemaname = 'public'
ORDER BY pg_relation_size(indexrelid) DESC;
```

---

### 3. What is a materialized view, and when should you use it?

**Materialized View:**

A materialized view is a database object that stores the result of a query physically. Unlike regular views (which are virtual), materialized views cache the query results on disk.

**Regular View vs Materialized View:**

```sql
-- Regular View (virtual, no storage)
CREATE VIEW user_stats AS
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(total) as total_spent
FROM orders
GROUP BY user_id;

-- Query always executes fresh
SELECT * FROM user_stats;  -- Runs the GROUP BY every time


-- Materialized View (physical storage)
CREATE MATERIALIZED VIEW user_stats_mv AS
SELECT 
    user_id,
    COUNT(*) as order_count,
    SUM(total) as total_spent
FROM orders
GROUP BY user_id;

-- Query reads pre-computed results
SELECT * FROM user_stats_mv;  -- Very fast! Just reads stored data
```

---

**How Materialized Views Work:**

1. **Creation**: Executes query and stores results
2. **Query**: Reads from stored results (fast)
3. **Refresh**: Re-executes query and updates stored results

```sql
-- Create materialized view
CREATE MATERIALIZED VIEW sales_summary AS
SELECT 
    DATE_TRUNC('day', order_date) as date,
    product_id,
    SUM(quantity) as total_quantity,
    SUM(quantity * price) as total_revenue
FROM order_items
JOIN orders USING (order_id)
GROUP BY date, product_id;

-- Add index to materialized view (makes queries even faster)
CREATE INDEX idx_sales_summary_date ON sales_summary(date);
CREATE INDEX idx_sales_summary_product ON sales_summary(product_id);

-- Query the materialized view (fast!)
SELECT * FROM sales_summary 
WHERE date >= '2024-01-01'
ORDER BY total_revenue DESC;
```

---

**Refreshing Materialized Views:**

**1. Complete Refresh (default):**
```sql
-- Refresh entire view (rebuilds all data)
REFRESH MATERIALIZED VIEW sales_summary;

-- Locks view during refresh - queries blocked
-- Use CONCURRENTLY to allow reads during refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;

-- Note: CONCURRENTLY requires a UNIQUE index
CREATE UNIQUE INDEX idx_sales_summary_unique 
ON sales_summary(date, product_id);
```

**2. Automatic Refresh with Triggers:**
```sql
-- Create function to refresh materialized view
CREATE OR REPLACE FUNCTION refresh_sales_summary()
RETURNS TRIGGER AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary;
    RETURN NULL;
END;
$$ LANGUAGE plpgsql;

-- Trigger on INSERT/UPDATE/DELETE
CREATE TRIGGER trigger_refresh_sales_summary
AFTER INSERT OR UPDATE OR DELETE ON orders
FOR EACH STATEMENT
EXECUTE FUNCTION refresh_sales_summary();

-- Warning: Can be expensive for frequent updates!
```

**3. Scheduled Refresh (using pg_cron):**
```sql
-- Install pg_cron extension
CREATE EXTENSION pg_cron;

-- Schedule refresh every hour
SELECT cron.schedule(
    'refresh-sales-summary',
    '0 * * * *',  -- Every hour
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary$$
);

-- Schedule daily at 2 AM
SELECT cron.schedule(
    'refresh-sales-summary-daily',
    '0 2 * * *',  -- 2 AM every day
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY sales_summary$$
);
```

---

**When to Use Materialized Views:**

**✅ Good Use Cases:**

**1. Expensive Aggregations:**
```sql
-- Query takes 30 seconds to run
CREATE MATERIALIZED VIEW customer_lifetime_value AS
SELECT 
    c.customer_id,
    c.customer_name,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(o.total) as total_spent,
    AVG(o.total) as avg_order_value,
    MAX(o.order_date) as last_order_date,
    EXTRACT(DAYS FROM NOW() - MAX(o.order_date)) as days_since_last_order
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN order_items oi ON o.order_id = oi.order_id
LEFT JOIN products p ON oi.product_id = p.product_id
GROUP BY c.customer_id, c.customer_name;

-- Now query runs in milliseconds instead of seconds
SELECT * FROM customer_lifetime_value
WHERE total_spent > 10000
ORDER BY total_spent DESC;
```

**2. Complex Joins:**
```sql
CREATE MATERIALIZED VIEW product_analytics AS
SELECT 
    p.product_id,
    p.product_name,
    c.category_name,
    COUNT(DISTINCT o.order_id) as times_ordered,
    SUM(oi.quantity) as total_sold,
    SUM(oi.quantity * oi.price) as total_revenue,
    AVG(r.rating) as avg_rating,
    COUNT(r.review_id) as review_count
FROM products p
JOIN categories c ON p.category_id = c.category_id
LEFT JOIN order_items oi ON p.product_id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.order_id
LEFT JOIN reviews r ON p.product_id = r.product_id
GROUP BY p.product_id, p.product_name, c.category_name;
```

**3. Dashboard / Reporting Queries:**
```sql
-- Daily dashboard data
CREATE MATERIALIZED VIEW daily_dashboard AS
SELECT 
    DATE_TRUNC('day', order_date) as date,
    COUNT(DISTINCT order_id) as order_count,
    COUNT(DISTINCT customer_id) as unique_customers,
    SUM(total) as revenue,
    AVG(total) as avg_order_value,
    COUNT(DISTINCT CASE WHEN status = 'completed' THEN order_id END) as completed_orders,
    COUNT(DISTINCT CASE WHEN status = 'cancelled' THEN order_id END) as cancelled_orders
FROM orders
GROUP BY date;

-- Refresh once per day
REFRESH MATERIALIZED VIEW daily_dashboard;
```

**4. Data Denormalization for Read Performance:**
```sql
CREATE MATERIALIZED VIEW order_details_denormalized AS
SELECT 
    o.order_id,
    o.order_date,
    o.status,
    c.customer_name,
    c.customer_email,
    c.customer_city,
    c.customer_country,
    p.product_name,
    p.category_name,
    oi.quantity,
    oi.price,
    oi.quantity * oi.price as line_total
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id;

-- No JOINs needed anymore
SELECT * FROM order_details_denormalized
WHERE order_date >= '2024-01-01';
```

---

**❌ When NOT to Use Materialized Views:**

**1. Frequently Changing Data:**
```sql
-- Bad: Real-time stock prices
CREATE MATERIALIZED VIEW stock_prices AS
SELECT * FROM stocks WHERE last_update > NOW() - INTERVAL '1 minute';
-- Data is stale immediately!
```

**2. Small, Fast Queries:**
```sql
-- Bad: Query is already fast
CREATE MATERIALIZED VIEW active_users AS
SELECT * FROM users WHERE is_active = true;
-- Just use a regular query or index
```

**3. Always Need Fresh Data:**
```sql
-- Bad: Need real-time balance
CREATE MATERIALIZED VIEW account_balances AS
SELECT account_id, SUM(amount) FROM transactions GROUP BY account_id;
-- Use regular view or real-time query
```

---

**Materialized View Comparison:**

| Aspect | Regular View | Materialized View | Table |
|--------|--------------|-------------------|-------|
| Storage | No (virtual) | Yes (physical) | Yes |
| Query Speed | Slow (runs query) | Fast (reads data) | Fast |
| Data Freshness | Always fresh | Stale until refresh | Depends |
| Maintenance | None | Manual/scheduled refresh | Manual |
| Indexes | No | Yes | Yes |
| Use Case | Simple transformations | Expensive queries | All data |

---

**Best Practices:**

```sql
-- 1. Always add indexes to materialized views
CREATE MATERIALIZED VIEW mv_example AS ...;
CREATE INDEX idx_mv_example_date ON mv_example(date);
CREATE INDEX idx_mv_example_category ON mv_example(category);

-- 2. Use CONCURRENTLY for production (requires UNIQUE index)
CREATE UNIQUE INDEX idx_mv_unique ON mv_example(id);
REFRESH MATERIALIZED VIEW CONCURRENTLY mv_example;

-- 3. Monitor view staleness
SELECT 
    schemaname,
    matviewname,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||matviewname)) as size,
    CASE WHEN last_refresh IS NOT NULL 
        THEN NOW() - last_refresh 
        ELSE NULL 
    END as age
FROM pg_matviews;

-- 4. Consider partial materialized views
CREATE MATERIALIZED VIEW recent_sales AS
SELECT * FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '90 days';
-- Smaller, faster to refresh

-- 5. Drop and recreate vs refresh for major schema changes
DROP MATERIALIZED VIEW IF EXISTS old_mv;
CREATE MATERIALIZED VIEW new_mv AS ...;
```

---

**Real-World Example: E-commerce Analytics**

```sql
-- Problem: Complex analytics query takes 45 seconds
-- Solution: Materialized view refreshed hourly

CREATE MATERIALIZED VIEW product_performance AS
WITH 
product_sales AS (
    SELECT 
        product_id,
        DATE_TRUNC('day', order_date) as date,
        SUM(quantity) as units_sold,
        SUM(quantity * price) as revenue
    FROM orders o
    JOIN order_items oi USING (order_id)
    WHERE order_date >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY product_id, date
),
product_reviews AS (
    SELECT 
        product_id,
        AVG(rating) as avg_rating,
        COUNT(*) as review_count
    FROM reviews
    WHERE created_at >= CURRENT_DATE - INTERVAL '90 days'
    GROUP BY product_id
)
SELECT 
    p.product_id,
    p.product_name,
    p.category,
    COALESCE(SUM(ps.units_sold), 0) as total_units_sold,
    COALESCE(SUM(ps.revenue), 0) as total_revenue,
    pr.avg_rating,
    pr.review_count,
    RANK() OVER (PARTITION BY p.category ORDER BY SUM(ps.revenue) DESC) as category_rank
FROM products p
LEFT JOIN product_sales ps USING (product_id)
LEFT JOIN product_reviews pr USING (product_id)
GROUP BY p.product_id, p.product_name, p.category, pr.avg_rating, pr.review_count;

-- Add indexes
CREATE INDEX idx_product_perf_category ON product_performance(category);
CREATE UNIQUE INDEX idx_product_perf_id ON product_performance(product_id);

-- Query is now instant instead of 45 seconds
SELECT * FROM product_performance
WHERE category = 'Electronics'
ORDER BY total_revenue DESC
LIMIT 10;

-- Refresh hourly
SELECT cron.schedule('refresh-product-performance', '0 * * * *',
    $$REFRESH MATERIALIZED VIEW CONCURRENTLY product_performance$$);
```

---

**Summary:**

- **Use materialized views** for expensive queries on relatively stable data
- **Refresh strategy** depends on data change frequency and freshness requirements
- **Add indexes** to materialized views for optimal query performance
- **Monitor staleness** and adjust refresh schedule as needed
- **Consider alternatives** like regular views with indexes or incremental updates for frequently changing data

---

###4. Explain query planner and `EXPLAIN ANALYZE`.

(This was already covered in detail in the Fundamentals section, but I'll provide a PostgreSQL-specific version)

The **Query Planner** (also called Query Optimizer) in PostgreSQL determines the most efficient way to execute a SQL query by evaluating different execution strategies and choosing the one with the lowest estimated cost.

**How the Query Planner Works:**

1. **Parse** the SQL query
2. **Rewrite** the query (apply rules, expand views)
3. **Plan** - Generate possible execution plans
4. **Optimize** - Choose the lowest-cost plan
5. **Execute** the chosen plan

---

**EXPLAIN - Show the Execution Plan:**

```sql
EXPLAIN SELECT * FROM users WHERE email = 'user@example.com';
```

Output shows the planned execution strategy without running the query.

---

**EXPLAIN ANALYZE - Run and Show Actual Performance:**

```sql
EXPLAIN ANALYZE SELECT * FROM users WHERE email = 'user@example.com';
```

Executes the query and shows:
- Planned cost estimates
- **Actual execution time**
- **Actual rows returned**
- Buffer usage
- Planning time vs execution time

---

**EXPLAIN Options:**

```sql
-- Basic explain
EXPLAIN SELECT ...;

-- Execute and show actual times
EXPLAIN ANALYZE SELECT ...;

-- Show more details
EXPLAIN (ANALYZE, BUFFERS, VERBOSE) SELECT ...;

-- JSON format (easier for tools to parse)
EXPLAIN (FORMAT JSON, ANALYZE) SELECT ...;

-- Other formats
EXPLAIN (FORMAT YAML) SELECT ...;
EXPLAIN (FORMAT XML) SELECT ...;
```

---

**Reading EXPLAIN Output:**

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE user_id = 12345;

/*
Index Scan using idx_users_id on users  
  (cost=0.42..8.44 rows=1 width=128) 
  (actual time=0.015..0.016 rows=1 loops=1)
  Index Cond: (user_id = 12345)
Planning Time: 0.123 ms
Execution Time: 0.045 ms
*/
```

**Understanding the Output:**

1. **Index Scan using idx_users_id** - Scan type and index used
2. **cost=0.42..8.44** - Estimated cost (startup..total)
3. **rows=1** - Estimated rows returned
4. **width=128** - Average row size in bytes
5. **actual time=0.015..0.016** - Actual time (first row..last row) in ms
6. **rows=1** - Actual rows returned
7. **loops=1** - How many times this node executed

---

**Common Scan Types:**

**1. Sequential Scan:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE city = 'New York';

/*
Seq Scan on users  (cost=0.00..1693.00 rows=50000 width=128)
  Filter: (city = 'New York')
  Rows Removed by Filter: 950000
*/
-- Reads entire table row by row
-- SLOW for large tables
```

**2. Index Scan:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE user_id = 123;

/*
Index Scan using idx_users_id on users  (cost=0.42..8.44 rows=1)
  Index Cond: (user_id = 123)
*/
-- Uses index to find rows
-- FAST
```

**3. Index Only Scan:**
```sql
-- All required columns are in the index
EXPLAIN ANALYZE
SELECT user_id, email FROM users WHERE user_id = 123;

/*
Index Only Scan using idx_users_id_email on users  (cost=0.42..4.44 rows=1)
  Index Cond: (user_id = 123)
  Heap Fetches: 0
*/
-- Doesn't need to access table at all
-- FASTEST
```

**4. Bitmap Heap Scan:**
```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE age > 25 AND city = 'NYC';

/*
Bitmap Heap Scan on users  (cost=104.32..356.78 rows=500)
  Recheck Cond: ((age > 25) AND (city = 'NYC'))
  ->  BitmapAnd
        ->  Bitmap Index Scan on idx_users_age
              Index Cond: (age > 25)
        ->  Bitmap Index Scan on idx_users_city
              Index Cond: (city = 'NYC')
*/
-- Combines multiple indexes
-- Good for multiple conditions
```

---

**Join Methods:**

**1. Nested Loop:**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id
WHERE c.customer_id = 123;

/*
Nested Loop  (cost=0.85..24.90 rows=5)
  ->  Index Scan using idx_customers_id on customers c
  ->  Index Scan using idx_orders_customer on orders o
        Index Cond: (customer_id = c.customer_id)
*/
-- Good for small result sets
-- Fast startup time
```

**2. Hash Join:**
```sql
EXPLAIN ANALYZE
SELECT * FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

/*
Hash Join  (cost=1234.56..5678.90 rows=100000)
  Hash Cond: (o.customer_id = c.customer_id)
  ->  Seq Scan on orders o
  ->  Hash
        ->  Seq Scan on customers c
*/
-- Good for large result sets
-- Higher startup cost (builds hash table)
```

**3. Merge Join:**
```sql
/*
Merge Join  (cost=245.32..1234.56 rows=10000)
  Merge Cond: (o.customer_id = c.customer_id)
  ->  Index Scan using idx_orders_customer on orders o
  ->  Index Scan using idx_customers_id on customers c
*/
-- Both inputs must be sorted
-- Good for large sorted datasets
```

---

**Analyzing Performance Issues:**

**Example 1: Missing Index**

```sql
-- Slow query
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;

/*
Seq Scan on orders  (cost=0.00..18334.00 rows=5) (actual time=234.567..1245.890 rows=5)
  Filter: (customer_id = 123)
  Rows Removed by Filter: 999995
Planning Time: 0.234 ms
Execution Time: 1246.123 ms  ← SLOW!
*/

-- Add index
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Now fast
EXPLAIN ANALYZE
SELECT * FROM orders WHERE customer_id = 123;

/*
Index Scan using idx_orders_customer on orders  (cost=0.42..12.45 rows=5) (actual time=0.034..0.045 rows=5)
  Index Cond: (customer_id = 123)
Planning Time: 0.156 ms
Execution Time: 0.067 ms  ← FAST!
*/
```

**Example 2: Function on Indexed Column**

```sql
-- Index not used
EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

/*
Seq Scan on users  (cost=0.00..18334.00 rows=5000)
  Filter: (LOWER(email) = 'user@example.com')
*/
-- Function prevents index usage

-- Solution: Functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));

-- Now uses index
EXPLAIN ANALYZE
SELECT * FROM users WHERE LOWER(email) = 'user@example.com';

/*
Index Scan using idx_users_email_lower on users  (cost=0.42..8.44 rows=1)
  Index Cond: (LOWER(email) = 'user@example.com')
*/
```

**Example 3: Outdated Statistics**

```sql
-- Query planner uses outdated statistics
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'completed';

/*
Estimated rows: 1000
Actual rows: 500000  ← Big difference!
*/

-- Update statistics
ANALYZE orders;

-- Now estimates are accurate
```

---

**Buffer Usage Analysis:**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM users WHERE user_id = 123;

/*
Index Scan using idx_users_id on users  (actual time=0.015..0.016 rows=1)
  Index Cond: (user_id = 123)
  Buffers: shared hit=4
Planning:
  Buffers: shared hit=12
Planning Time: 0.123 ms
Execution Time: 0.045 ms
*/
```

**Buffer Types:**
- **shared hit** - Found in PostgreSQL's shared buffer cache (RAM) ✅ Good
- **shared read** - Had to read from disk ❌ Slower
- **temp read/written** - Used temporary files (ran out of work_mem) ❌ Bad

---

**Query Planner Configuration:**

```sql
-- View current planner settings
SHOW all;

-- Important settings:
SHOW work_mem;              -- Memory for sorting, hashing
SHOW shared_buffers;        -- PostgreSQL cache size
SHOW effective_cache_size;  -- Estimate of OS cache
SHOW random_page_cost;      -- Cost of random disk access
SHOW seq_page_cost;         -- Cost of sequential disk access

-- Adjust for specific query
SET work_mem = '256MB';
EXPLAIN ANALYZE SELECT ...;
RESET work_mem;

-- Disable specific scan types (for testing)
SET enable_seqscan = OFF;
SET enable_hashjoin = OFF;
```

---

**Practical Workflow:**

```sql
-- 1. Find slow queries
SELECT query, mean_exec_time, calls
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- 2. Analyze the query
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;

-- 3. Look for issues:
--    - Sequential scans on large tables
--    - High "Rows Removed by Filter"
--    - Missing indexes
--    - Estimated vs actual row mismatch
--    - High buffer reads (not hits)

-- 4. Fix issues:
--    - Add indexes
--    - Update statistics (ANALYZE)
--    - Rewrite query
--    - Adjust work_mem

-- 5. Verify improvement
EXPLAIN (ANALYZE, BUFFERS)
SELECT ...;
```

---

### 5. Difference between sequential scan and index scan.

**Sequential Scan (Seq Scan):**
- Reads table from beginning to end
- Checks every row
- No index used

**Index Scan:**
- Uses index to find specific rows
- Only reads matching rows
- Requires index to exist

---

**Sequential Scan Example:**

```sql
EXPLAIN ANALYZE
SELECT * FROM users WHERE city = 'New York';

/*
Seq Scan on users  (cost=0.00..18334.00 rows=50000 width=128) 
                   (actual time=0.012..245.678 rows=50000 loops=1)
  Filter: (city = 'New York')
  Rows Removed by Filter: 950000
  Buffers: shared hit=8000  ← Read entire table
*/
```

**How it works:**
1. Start at first row
2. Read each row sequentially
3. Check if row matches condition
4. Return matching rows

**Characteristics:**
- ✅ Good for: Reading entire table or large % of rows
- ✅ Good for: Small tables
- ❌ Bad for: Finding few rows in large table
- ❌ Bad for: Selective queries

---

**Index Scan Example:**

```sql
-- Create index
CREATE INDEX idx_users_city ON users(city);

EXPLAIN ANALYZE
SELECT * FROM users WHERE city = 'New York';

/*
Index Scan using idx_users_city on users  (cost=0.42..1234.56 rows=50000 width=128)
                                          (actual time=0.025..45.123 rows=50000 loops=1)
  Index Cond: (city = 'New York')
  Buffers: shared hit=1200  ← Read only index + matching rows
*/
```

**How it works:**
1. Traverse B-tree index to find starting point
2. Follow index to find matching rows
3. Fetch actual row data from table
4. Return results

**Characteristics:**
- ✅ Good for: Finding few rows
- ✅ Good for: Selective queries
- ❌ Bad for: Reading most of table (slower than seq scan!)
- Requires index maintenance overhead

---

**When PostgreSQL Chooses Each:**

```sql
-- Example table: 1,000,000 users
-- 100,000 in New York (10%)
-- 900,000 elsewhere (90%)

-- Query 1: Returns 10% of rows
SELECT * FROM users WHERE city = 'New York';
-- Planner might choose: Sequential Scan
-- Reason: Reading 10% = need to access many random pages
--         Sequential scan might be faster

-- Query 2: Returns 0.001% of rows
SELECT * FROM users WHERE user_id = 12345;
-- Planner chooses: Index Scan
-- Reason: Only need 1 row, index is much faster

-- Query 3: Returns 90% of rows
SELECT * FROM users WHERE city != 'New York';
-- Planner chooses: Sequential Scan
-- Reason: Need most of table anyway, seq scan is faster
```

---

**Cost Calculation (Simplified):**

```sql
-- Sequential Scan Cost:
seq_scan_cost = (pages_to_read * seq_page_cost) + (rows_to_check * cpu_tuple_cost)
              = (10000 pages * 1.0) + (1000000 rows * 0.01)
              = 10000 + 10000 = 20,000

-- Index Scan Cost (for 100 rows):
index_scan_cost = (index_pages * random_page_cost) + 
                  (matching_rows * random_page_cost) + 
                  (cpu costs)
                = (5 pages * 4.0) + (100 rows * 4.0) + (small cpu cost)
                = 20 + 400 + 1 = 421

-- If matching_rows = 100: Index Scan wins (421 < 20,000)
-- If matching_rows = 100,000: Sequential Scan wins
```

---

**Selectivity Matters:**

```sql
-- High selectivity (few rows) → Index Scan
SELECT * FROM orders WHERE order_id = 12345;
-- 1 out of 1,000,000 rows = 0.0001% selectivity
-- Index Scan: ~5-10 ms

-- Low selectivity (many rows) → Sequential Scan
SELECT * FROM orders WHERE status = 'completed';
-- 900,000 out of 1,000,000 rows = 90% selectivity
-- Sequential Scan: ~200 ms
-- Index Scan would be: ~5000 ms (slower!)
```

---

**Forcing Scan Type (for testing):**

```sql
-- Force sequential scan
SET enable_indexscan = OFF;
EXPLAIN ANALYZE SELECT * FROM users WHERE user_id = 123;
RESET enable_indexscan;

-- Force index scan
SET enable_seqscan = OFF;
EXPLAIN ANALYZE SELECT * FROM users WHERE city = 'New York';
RESET enable_seqscan;
```

---

**Index-Only Scan (Bonus):**

Best of both worlds - reads only from index, doesn't touch table!

```sql
-- Create covering index
CREATE INDEX idx_users_id_email ON users(user_id, email);

EXPLAIN ANALYZE
SELECT user_id, email FROM users WHERE user_id = 123;

/*
Index Only Scan using idx_users_id_email on users  (cost=0.42..4.44 rows=1)
  Index Cond: (user_id = 123)
  Heap Fetches: 0  ← Didn't need to read table!
*/
-- FASTEST option!
```

---

**Comparison Table:**

| Aspect | Sequential Scan | Index Scan | Index Only Scan |
|--------|----------------|------------|-----------------|
| **Speed** (few rows) | Slow | Fast | Fastest |
| **Speed** (many rows) | Fast | Slow | Depends |
| **Disk I/O** | High | Medium | Low |
| **Requires Index** | No | Yes | Yes |
| **Random Access** | No | Yes | Minimal |
| **Best For** | >5-10% of rows | <5% of rows | Covered queries |

---

**Real-World Example:**

```sql
-- Table: 10 million orders

-- Query 1: Find one order (use index!)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_id = 12345;
-- Index Scan: 0.05 ms ✅
-- Seq Scan: 2500 ms ❌

-- Query 2: Get all orders (use seq scan!)
EXPLAIN ANALYZE
SELECT * FROM orders;
-- Seq Scan: 3000 ms ✅
-- Index Scan: Would never finish ❌

-- Query 3: Recent orders (1% of data - index!)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date >= CURRENT_DATE - 7;
-- Index Scan: 150 ms ✅
-- Seq Scan: 2500 ms ❌

-- Query 4: Old orders (99% of data - seq scan!)
EXPLAIN ANALYZE
SELECT * FROM orders WHERE order_date < CURRENT_DATE - 7;
-- Seq Scan: 2600 ms ✅
-- Index Scan: 8000 ms ❌
```

---

**Key Takeaways:**

1. **Index scan is NOT always faster** - depends on selectivity
2. **PostgreSQL chooses automatically** based on statistics
3. **Keep statistics up to date** - `ANALYZE` regularly
4. **Index-Only Scan is optimal** when possible
5. **Monitor query plans** with EXPLAIN ANALYZE
6. **Consider covering indexes** for frequently queried columns

---

I'll continue with the remaining questions in the next response to keep this manageable. Shall I continue with questions 6-8 of Advanced SQL Concepts and then the Performance & Scaling section?


---

### C. Performance & Scaling

**1. How do you tune queries for performance?**

Query performance tuning is a systematic process of identifying bottlenecks and optimizing SQL queries to execute faster. It involves analyzing query execution plans, optimizing indexes, rewriting queries, and adjusting database configuration.

**The Performance Tuning Process:**

1. **Identify slow queries** (monitoring)
2. **Analyze execution plans** (EXPLAIN ANALYZE)
3. **Find bottlenecks** (scans, joins, filters)
4. **Apply optimizations** (indexes, query rewrites)
5. **Verify improvements** (measure)
6. **Monitor continuously** (production)

---

**Step 1: Identify Slow Queries**

```sql
-- Enable pg_stat_statements extension
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

-- Find slowest queries by total time
SELECT 
    query,
    calls,
    total_exec_time,
    mean_exec_time,
    max_exec_time,
    stddev_exec_time,
    rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Find slowest queries by average time
SELECT 
    query,
    calls,
    ROUND(mean_exec_time::numeric, 2) as avg_ms,
    ROUND((total_exec_time/1000)::numeric, 2) as total_sec
FROM pg_stat_statements
WHERE mean_exec_time > 100  -- Queries taking more than 100ms
ORDER BY mean_exec_time DESC
LIMIT 20;

-- Find queries with high variance (unpredictable performance)
SELECT 
    query,
    calls,
    mean_exec_time,
    stddev_exec_time,
    (stddev_exec_time / mean_exec_time) as coefficient_of_variation
FROM pg_stat_statements
WHERE calls > 100
ORDER BY coefficient_of_variation DESC
LIMIT 10;
```

---

**Step 2: Analyze with EXPLAIN ANALYZE**

```sql
-- Get detailed execution plan
EXPLAIN (ANALYZE, BUFFERS, VERBOSE)
SELECT u.name, o.order_date, o.total_amount
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.city = 'New York'
  AND o.order_date >= '2024-01-01';

/*
Output shows:
1. Execution plan nodes
2. Estimated vs actual rows
3. Actual execution time
4. Buffer hits/reads
5. Join methods used
*/
```

**What to Look For:**

```sql
-- ❌ Problem 1: Sequential Scan on Large Table
Seq Scan on orders  (cost=0.00..180000.00 rows=50000)
  Filter: (order_date >= '2024-01-01')
  Rows Removed by Filter: 950000  ← Reading too many rows!

-- ✅ Solution: Add index
CREATE INDEX idx_orders_date ON orders(order_date);

-- ❌ Problem 2: Estimated vs Actual Mismatch
Hash Join  (cost=100.00..200.00 rows=10)  ← Estimated 10 rows
           (actual rows=50000 loops=1)     ← Actually 50,000 rows!

-- ✅ Solution: Update statistics
ANALYZE orders;

-- ❌ Problem 3: High "Rows Removed by Filter"
Filter: (city = 'New York' AND country = 'USA')
Rows Removed by Filter: 999000  ← Filtering 99.9% of rows!

-- ✅ Solution: Add composite index
CREATE INDEX idx_users_city_country ON users(city, country);

-- ❌ Problem 4: Nested Loop on Large Dataset
Nested Loop  (cost=0.00..1000000.00 rows=500000)
  -> Seq Scan on users  (rows=10000)
  -> Index Scan on orders  (rows=50)

-- ✅ Solution: Force hash join
SET enable_nestloop = OFF;  -- Test only
-- Or rewrite query to help planner choose hash join

-- ❌ Problem 5: External Sort (Disk Spill)
Sort  (actual time=5000.000..5200.000 rows=1000000)
  Sort Key: order_date
  Sort Method: external merge  Disk: 45MB  ← Spilled to disk!
  
-- ✅ Solution: Increase work_mem
SET work_mem = '256MB';
```

---

**Step 3: Index Optimization**

**Creating the Right Indexes:**

```sql
-- 1. Single column index (equality searches)
CREATE INDEX idx_users_email ON users(email);
-- Speeds up: WHERE email = 'user@example.com'

-- 2. Composite index (multiple columns)
CREATE INDEX idx_orders_user_date ON orders(user_id, order_date);
-- Speeds up: WHERE user_id = 123 AND order_date >= '2024-01-01'
-- Order matters! user_id first, then order_date

-- 3. Covering index (includes extra columns)
CREATE INDEX idx_orders_user_date_total ON orders(user_id, order_date) 
INCLUDE (total_amount, status);
-- Index-only scan possible for: 
-- SELECT total_amount, status WHERE user_id = ? AND order_date = ?

-- 4. Partial index (filtered index)
CREATE INDEX idx_orders_pending ON orders(created_at)
WHERE status = 'pending';
-- Smaller index, faster searches for pending orders
-- Speeds up: WHERE status = 'pending' AND created_at >= ...

-- 5. Expression/Functional index
CREATE INDEX idx_users_email_lower ON users(LOWER(email));
-- Speeds up: WHERE LOWER(email) = 'user@example.com'

-- 6. GIN index (for arrays, JSONB, full-text search)
CREATE INDEX idx_products_tags ON products USING GIN(tags);
-- Speeds up: WHERE tags @> ARRAY['electronics', 'sale']

CREATE INDEX idx_products_data ON products USING GIN(data jsonb_path_ops);
-- Speeds up: WHERE data @> '{"category": "electronics"}'

-- 7. BRIN index (for very large sequential data)
CREATE INDEX idx_logs_timestamp ON logs USING BRIN(timestamp);
-- Great for time-series data, minimal storage
```

**Index Maintenance:**

```sql
-- Find unused indexes
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as index_size
FROM pg_stat_user_indexes
WHERE idx_scan = 0
  AND indexrelname NOT LIKE 'pg_toast%'
ORDER BY pg_relation_size(indexrelid) DESC;

-- Find duplicate indexes
SELECT 
    pg_size_pretty(SUM(pg_relation_size(idx))::BIGINT) as size,
    (array_agg(idx))[1] as idx1,
    (array_agg(idx))[2] as idx2,
    (array_agg(idx))[3] as idx3,
    (array_agg(idx))[4] as idx4
FROM (
    SELECT 
        indexrelid::regclass as idx,
        (indrelid::text ||E'\n'|| indclass::text ||E'\n'|| 
         indkey::text ||E'\n'||COALESCE(indexprs::text,'')||E'\n' ||
         COALESCE(indpred::text,'')) as key
    FROM pg_index
) sub
GROUP BY key 
HAVING COUNT(*) > 1
ORDER BY SUM(pg_relation_size(idx)) DESC;

-- Reindex bloated indexes
REINDEX INDEX CONCURRENTLY idx_users_email;

-- Update index statistics
ANALYZE users;
```

---

**Step 4: Query Rewriting**

**Optimization Technique 1: Avoid Function Calls on Indexed Columns**

```sql
-- ❌ Slow: Function prevents index usage
SELECT * FROM orders 
WHERE EXTRACT(YEAR FROM order_date) = 2024;

-- ✅ Fast: Use index-friendly condition
SELECT * FROM orders 
WHERE order_date >= '2024-01-01' 
  AND order_date < '2025-01-01';
```

**Optimization Technique 2: Avoid SELECT ***

```sql
-- ❌ Slow: Fetches all columns (more I/O)
SELECT * FROM users WHERE user_id = 123;

-- ✅ Fast: Only fetch needed columns
SELECT user_id, name, email FROM users WHERE user_id = 123;

-- ✅ Even better: Index-only scan possible
CREATE INDEX idx_users_id_name_email ON users(user_id) INCLUDE (name, email);
```

**Optimization Technique 3: Use EXISTS Instead of IN with Subquery**

```sql
-- ❌ Slower: IN with subquery
SELECT * FROM users
WHERE user_id IN (
    SELECT user_id FROM orders WHERE order_date >= '2024-01-01'
);

-- ✅ Faster: EXISTS (can short-circuit)
SELECT * FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.user_id
      AND o.order_date >= '2024-01-01'
);

-- ✅ Best: JOIN (if you need order data)
SELECT DISTINCT u.*
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE o.order_date >= '2024-01-01';
```

**Optimization Technique 4: Break Complex Queries into CTEs**

```sql
-- ❌ Complex nested query (hard to optimize)
SELECT *
FROM users
WHERE user_id IN (
    SELECT user_id FROM orders
    WHERE order_id IN (
        SELECT order_id FROM order_items
        WHERE product_id IN (
            SELECT product_id FROM products WHERE category = 'Electronics'
        )
    )
);

-- ✅ Clearer with CTEs (planner can optimize better)
WITH electronic_products AS (
    SELECT product_id FROM products WHERE category = 'Electronics'
),
relevant_order_items AS (
    SELECT DISTINCT order_id 
    FROM order_items oi
    JOIN electronic_products ep ON oi.product_id = ep.product_id
),
relevant_orders AS (
    SELECT DISTINCT user_id
    FROM orders o
    JOIN relevant_order_items roi ON o.order_id = roi.order_id
)
SELECT u.*
FROM users u
JOIN relevant_orders ro ON u.user_id = ro.user_id;
```

**Optimization Technique 5: Use LIMIT with ORDER BY**

```sql
-- ❌ Slow: Sorts entire result set
SELECT * FROM orders
ORDER BY order_date DESC;

-- ✅ Fast: Planner can optimize with LIMIT
SELECT * FROM orders
ORDER BY order_date DESC
LIMIT 100;

-- ✅ Even better: Index supports the sort
CREATE INDEX idx_orders_date_desc ON orders(order_date DESC);
```

**Optimization Technique 6: Partition Large Queries**

```sql
-- ❌ Slow: Processing millions of rows
UPDATE orders 
SET status = 'archived'
WHERE order_date < '2020-01-01';
-- Locks table for long time

-- ✅ Fast: Batch updates
DO $$
DECLARE
    batch_size INT := 10000;
    affected INT;
BEGIN
    LOOP
        UPDATE orders 
        SET status = 'archived'
        WHERE order_id IN (
            SELECT order_id FROM orders
            WHERE order_date < '2020-01-01'
              AND status != 'archived'
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS affected = ROW_COUNT;
        EXIT WHEN affected = 0;
        
        COMMIT;  -- Release locks
        PERFORM pg_sleep(0.1);  -- Breathing room
    END LOOP;
END $$;
```

---

**Step 5: Join Optimization**

```sql
-- Understand join methods
EXPLAIN ANALYZE
SELECT u.name, o.order_date
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.city = 'New York';

/*
Three join methods:
1. Nested Loop - Good for small datasets or high selectivity
2. Hash Join - Good for medium to large datasets
3. Merge Join - Good for pre-sorted data
*/

-- Force specific join method (testing only)
SET enable_nestloop = OFF;
SET enable_hashjoin = OFF;
SET enable_mergejoin = OFF;
```

**Join Optimization Tips:**

```sql
-- 1. Filter early (push down predicates)
-- ❌ Bad: Filter after join
SELECT u.name, o.total_amount
FROM users u
JOIN orders o ON u.user_id = o.user_id
WHERE u.city = 'New York';

-- ✅ Good: Filter before join (if possible)
WITH nyc_users AS (
    SELECT user_id, name FROM users WHERE city = 'New York'
)
SELECT nu.name, o.total_amount
FROM nyc_users nu
JOIN orders o ON nu.user_id = o.user_id;

-- 2. Join smaller tables first
-- ✅ Good: Join order matters for nested loops
FROM small_table s
JOIN medium_table m ON s.id = m.small_id
JOIN large_table l ON m.id = l.medium_id

-- 3. Add indexes on join columns
CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_users_id ON users(user_id);  -- Usually PK already indexed
```

---

**Step 6: Configuration Tuning**

```sql
-- Memory settings (adjust based on server RAM)

-- shared_buffers: PostgreSQL cache (25% of RAM)
ALTER SYSTEM SET shared_buffers = '4GB';

-- work_mem: Per-operation memory (sorting, hashing)
ALTER SYSTEM SET work_mem = '64MB';  -- Be careful, per operation!

-- maintenance_work_mem: For VACUUM, CREATE INDEX
ALTER SYSTEM SET maintenance_work_mem = '1GB';

-- effective_cache_size: Estimate of OS cache (50-75% of RAM)
ALTER SYSTEM SET effective_cache_size = '12GB';

-- Parallelism settings
ALTER SYSTEM SET max_parallel_workers_per_gather = 4;
ALTER SYSTEM SET max_parallel_workers = 8;
ALTER SYSTEM SET parallel_tuple_cost = 0.1;

-- Query planner settings (for SSD)
ALTER SYSTEM SET random_page_cost = 1.1;  -- Default is 4.0 for HDD
ALTER SYSTEM SET seq_page_cost = 1.0;

-- Reload configuration
SELECT pg_reload_conf();
```

**Session-level tuning (for specific queries):**

```sql
-- Increase memory for a heavy query
SET work_mem = '512MB';
SELECT ... -- Heavy aggregation
RESET work_mem;

-- Disable parallel query (if causing issues)
SET max_parallel_workers_per_gather = 0;
SELECT ... -- Your query
RESET max_parallel_workers_per_gather;

-- Force planner to prefer index scans
SET random_page_cost = 1.0;
SELECT ... -- Your query
RESET random_page_cost;
```

---

**Step 7: Advanced Techniques**

**Table Partitioning:**

```sql
-- Partition by range (time-based)
CREATE TABLE orders (
    order_id BIGSERIAL,
    order_date DATE NOT NULL,
    user_id INT,
    total_amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

CREATE TABLE orders_2024_q2 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Queries automatically use partition pruning
EXPLAIN SELECT * FROM orders 
WHERE order_date >= '2024-01-01' AND order_date < '2024-02-01';
-- Only scans orders_2024_q1 partition!
```

**Materialized Views:**

```sql
-- Pre-compute expensive aggregations
CREATE MATERIALIZED VIEW daily_sales_summary AS
SELECT 
    DATE_TRUNC('day', order_date) as day,
    COUNT(*) as order_count,
    SUM(total_amount) as total_revenue,
    AVG(total_amount) as avg_order_value
FROM orders
GROUP BY DATE_TRUNC('day', order_date);

CREATE INDEX idx_daily_sales_day ON daily_sales_summary(day);

-- Refresh periodically
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_sales_summary;
```

**Query Result Caching:**

```sql
-- Application-level caching with expiry
-- Example: Cache in Redis for 5 minutes

-- Database-level: Prepared statements (plan caching)
PREPARE get_user_orders AS
SELECT * FROM orders WHERE user_id = $1;

EXECUTE get_user_orders(123);  -- Uses cached plan
```

---

**Monitoring and Continuous Optimization**

```sql
-- Monitor query performance over time
CREATE TABLE query_performance_log (
    query_hash TEXT,
    query_text TEXT,
    calls BIGINT,
    mean_exec_time FLOAT,
    total_exec_time FLOAT,
    recorded_at TIMESTAMP DEFAULT NOW()
);

-- Log daily performance
INSERT INTO query_performance_log
SELECT 
    md5(query) as query_hash,
    query,
    calls,
    mean_exec_time,
    total_exec_time,
    NOW()
FROM pg_stat_statements
WHERE mean_exec_time > 100;

-- Find queries that got slower
SELECT 
    curr.query_text,
    prev.mean_exec_time as old_avg_ms,
    curr.mean_exec_time as new_avg_ms,
    ROUND(((curr.mean_exec_time - prev.mean_exec_time) / 
           prev.mean_exec_time * 100)::numeric, 2) as pct_slower
FROM query_performance_log curr
JOIN query_performance_log prev 
    ON curr.query_hash = prev.query_hash
WHERE curr.recorded_at = CURRENT_DATE
  AND prev.recorded_at = CURRENT_DATE - 1
  AND curr.mean_exec_time > prev.mean_exec_time * 1.2  -- 20% slower
ORDER BY pct_slower DESC;

-- Monitor cache hit ratio
SELECT 
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as cache_hit_ratio
FROM pg_statio_user_tables;
-- Target: > 0.99 (99% cache hit ratio)

-- Monitor index usage
SELECT 
    schemaname,
    tablename,
    indexname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC
LIMIT 20;
```

---

**Real-World Example: E-commerce Query Optimization**

```sql
-- ❌ Original query: 8 seconds
SELECT 
    u.name,
    u.email,
    COUNT(DISTINCT o.order_id) as order_count,
    SUM(o.total_amount) as lifetime_value,
    MAX(o.order_date) as last_order_date,
    AVG(r.rating) as avg_rating
FROM users u
LEFT JOIN orders o ON u.user_id = o.user_id
LEFT JOIN reviews r ON u.user_id = r.user_id
WHERE u.created_at >= '2023-01-01'
  AND (o.status = 'completed' OR o.status IS NULL)
GROUP BY u.user_id, u.name, u.email
HAVING COUNT(DISTINCT o.order_id) > 5
ORDER BY lifetime_value DESC
LIMIT 100;

-- Analysis with EXPLAIN ANALYZE showed:
-- 1. Sequential scan on users table (10M rows)
-- 2. No index on orders.status
-- 3. Computing AVG(rating) for all users even though we only need top 100

-- ✅ Optimized query: 0.3 seconds (26x faster!)

-- Step 1: Add indexes
CREATE INDEX idx_users_created ON users(created_at);
CREATE INDEX idx_orders_user_status ON orders(user_id, status) 
    INCLUDE (total_amount, order_date);
CREATE INDEX idx_reviews_user ON reviews(user_id);

-- Step 2: Rewrite query to filter early
WITH valuable_users AS (
    SELECT 
        u.user_id,
        u.name,
        u.email
    FROM users u
    WHERE u.created_at >= '2023-01-01'
),
user_orders AS (
    SELECT 
        o.user_id,
        COUNT(DISTINCT o.order_id) as order_count,
        SUM(o.total_amount) as lifetime_value,
        MAX(o.order_date) as last_order_date
    FROM orders o
    WHERE o.status = 'completed'
      AND EXISTS (SELECT 1 FROM valuable_users vu WHERE vu.user_id = o.user_id)
    GROUP BY o.user_id
    HAVING COUNT(DISTINCT o.order_id) > 5
),
top_users AS (
    SELECT user_id, order_count, lifetime_value, last_order_date
    FROM user_orders
    ORDER BY lifetime_value DESC
    LIMIT 100
)
SELECT 
    vu.name,
    vu.email,
    tu.order_count,
    tu.lifetime_value,
    tu.last_order_date,
    AVG(r.rating) as avg_rating
FROM top_users tu
JOIN valuable_users vu ON tu.user_id = vu.user_id
LEFT JOIN reviews r ON tu.user_id = r.user_id
GROUP BY vu.name, vu.email, tu.order_count, tu.lifetime_value, tu.last_order_date
ORDER BY tu.lifetime_value DESC;
```

---

**Performance Tuning Checklist:**

- [ ] Enable and monitor `pg_stat_statements`
- [ ] Run `EXPLAIN ANALYZE` on slow queries
- [ ] Check for missing indexes on WHERE/JOIN columns
- [ ] Verify index usage with `pg_stat_user_indexes`
- [ ] Look for sequential scans on large tables
- [ ] Check estimated vs actual row counts (outdated statistics?)
- [ ] Run `ANALYZE` to update table statistics
- [ ] Remove unused indexes
- [ ] Ensure `shared_buffers` is 25% of RAM
- [ ] Set `effective_cache_size` to 50-75% of RAM
- [ ] Adjust `work_mem` for heavy sorting/aggregation
- [ ] Use `random_page_cost = 1.1` for SSDs
- [ ] Monitor cache hit ratio (target: >99%)
- [ ] Consider partitioning for large tables (>10GB)
- [ ] Use materialized views for expensive aggregations
- [ ] Batch large updates/deletes
- [ ] Monitor query performance trends

---

**Key Takeaways:**

1. **Always measure** - Use EXPLAIN ANALYZE before and after optimization
2. **Indexes are crucial** - But don't over-index (write performance trade-off)
3. **Statistics matter** - Run ANALYZE regularly to keep planner informed
4. **Query rewriting** often gives bigger gains than configuration tuning
5. **Consider the data distribution** - What works for 1M rows may not work for 100M
6. **Monitor continuously** - Performance degrades over time as data grows
7. **Optimize for the common case** - Focus on frequently-run queries
8. **Know your workload** - OLTP vs OLAP require different optimization strategies

---


**2. How do you handle millions of inserts/updates efficiently?**

Handling millions of inserts/updates efficiently requires using bulk operations, optimizing transactions, managing indexes strategically, and leveraging PostgreSQL-specific features. The key is to minimize overhead and maximize throughput.

---

**The Efficiency Hierarchy (Fastest to Slowest):**

1. **COPY command** - Fastest (binary protocol)
2. **Multi-row INSERT** - Very fast
3. **Prepared statements with batching** - Fast
4. **Individual INSERTs in transaction** - Slow
5. **Individual auto-commit INSERTs** - Extremely slow ❌

---

**Method 1: COPY Command (Fastest)**

The `COPY` command is PostgreSQL's fastest way to load data - up to 10-100x faster than INSERTs.

```sql
-- COPY from CSV file
COPY users(user_id, name, email, created_at)
FROM '/path/to/users.csv'
WITH (FORMAT csv, HEADER true, DELIMITER ',');

-- COPY from program/pipe
COPY users(user_id, name, email, created_at)
FROM PROGRAM 'gzip -dc /path/to/users.csv.gz'
WITH (FORMAT csv, HEADER true);

-- COPY from stdin (for programmatic use)
COPY users(user_id, name, email, created_at)
FROM stdin
WITH (FORMAT csv);
-- Send data through connection
-- End with \. on a line by itself
```

**Python Example with psycopg2:**

```python
import psycopg2
from io import StringIO

conn = psycopg2.connect("dbname=mydb user=postgres")
cur = conn.cursor()

# Prepare data in memory
data = StringIO()
for i in range(1000000):
    data.write(f"{i},User{i},user{i}@example.com,2024-01-01\n")
data.seek(0)

# Bulk insert using COPY
cur.copy_from(
    data,
    'users',
    columns=('user_id', 'name', 'email', 'created_at'),
    sep=','
)

conn.commit()
cur.close()
conn.close()

# Result: 1M rows inserted in ~2-5 seconds
```

**Python Example with copy_expert (more control):**

```python
import psycopg2

conn = psycopg2.connect("dbname=mydb user=postgres")
cur = conn.cursor()

# Use copy_expert for CSV with special characters
with open('users.csv', 'r') as f:
    cur.copy_expert(
        """
        COPY users(user_id, name, email, created_at)
        FROM STDIN
        WITH (FORMAT csv, HEADER true, QUOTE '"', ESCAPE '\\')
        """,
        f
    )

conn.commit()
```

---

**Method 2: Multi-Row INSERT**

Insert multiple rows in a single INSERT statement - much faster than individual INSERTs.

```sql
-- ❌ Slow: Individual INSERTs (1000 statements)
INSERT INTO users VALUES (1, 'User1', 'user1@example.com');
INSERT INTO users VALUES (2, 'User2', 'user2@example.com');
-- ... 998 more statements
-- Time: ~5 seconds for 1000 rows

-- ✅ Fast: Multi-row INSERT (1 statement)
INSERT INTO users (user_id, name, email) VALUES
    (1, 'User1', 'user1@example.com'),
    (2, 'User2', 'user2@example.com'),
    (3, 'User3', 'user3@example.com'),
    -- ... up to 1000 rows
    (1000, 'User1000', 'user1000@example.com');
-- Time: ~0.1 seconds for 1000 rows
```

**Python Example with executemany:**

```python
import psycopg2
import psycopg2.extras

conn = psycopg2.connect("dbname=mydb user=postgres")
cur = conn.cursor()

# Prepare data
data = [(i, f'User{i}', f'user{i}@example.com') for i in range(1000000)]

# ❌ Slow: Regular executemany (creates individual INSERTs)
# cur.executemany(
#     "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
#     data
# )
# Time: ~2 minutes for 1M rows

# ✅ Fast: execute_batch (batches into multi-row INSERTs)
psycopg2.extras.execute_batch(
    cur,
    "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
    data,
    page_size=1000  # 1000 rows per INSERT
)
# Time: ~10-15 seconds for 1M rows

# ✅ Even faster: execute_values (optimized for VALUES clause)
psycopg2.extras.execute_values(
    cur,
    "INSERT INTO users (user_id, name, email) VALUES %s",
    data,
    page_size=1000
)
# Time: ~8-12 seconds for 1M rows

conn.commit()
```

---

**Method 3: Transaction Management**

Wrap bulk operations in transactions to avoid commit overhead.

```sql
-- ❌ Terrible: Auto-commit for each INSERT
-- Each INSERT commits immediately = 1 million commits!
INSERT INTO users VALUES (1, 'User1', 'user1@example.com');  -- Commit!
INSERT INTO users VALUES (2, 'User2', 'user2@example.com');  -- Commit!
-- ... 999,998 more
-- Time: Could take hours!

-- ✅ Good: Single transaction
BEGIN;
    INSERT INTO users VALUES (1, 'User1', 'user1@example.com');
    INSERT INTO users VALUES (2, 'User2', 'user2@example.com');
    -- ... 999,998 more individual INSERTs
COMMIT;
-- Time: ~30 seconds for 1M rows

-- ✅ Better: Transaction with multi-row INSERTs
BEGIN;
    INSERT INTO users VALUES (1, 'User1', 'user1@example.com'), (2, 'User2', 'user2@example.com'), ... ; -- 1000 rows
    INSERT INTO users VALUES (1001, 'User1001', 'user1001@example.com'), ... ; -- 1000 rows
    -- ... 998 more statements
COMMIT;
-- Time: ~10 seconds for 1M rows
```

**Optimal Batch Size:**

```python
import psycopg2
import psycopg2.extras

def bulk_insert_optimized(data, batch_size=10000):
    """
    Insert data in batches with periodic commits
    Balances transaction size vs. rollback risk
    """
    conn = psycopg2.connect("dbname=mydb user=postgres")
    cur = conn.cursor()
    
    total_rows = len(data)
    
    for i in range(0, total_rows, batch_size):
        batch = data[i:i + batch_size]
        
        try:
            psycopg2.extras.execute_values(
                cur,
                "INSERT INTO users (user_id, name, email) VALUES %s",
                batch,
                page_size=1000
            )
            conn.commit()
            
            print(f"Inserted {min(i + batch_size, total_rows)}/{total_rows} rows")
            
        except Exception as e:
            conn.rollback()
            print(f"Error in batch {i}-{i+batch_size}: {e}")
            # Optionally: retry with smaller batches or log failed rows
    
    cur.close()
    conn.close()

# Usage
data = [(i, f'User{i}', f'user{i}@example.com') for i in range(1000000)]
bulk_insert_optimized(data, batch_size=10000)
# Time: ~8-10 seconds for 1M rows
```

---

**Method 4: Disable/Defer Constraints and Triggers**

Temporarily disable constraints and triggers during bulk operations.

```sql
-- Scenario: Loading 10M rows into a table with multiple indexes and foreign keys

BEGIN;

-- 1. Disable triggers (if any)
ALTER TABLE orders DISABLE TRIGGER ALL;

-- 2. Drop non-essential indexes temporarily
DROP INDEX IF EXISTS idx_orders_date;
DROP INDEX IF EXISTS idx_orders_user;
DROP INDEX IF EXISTS idx_orders_status;
-- Keep PRIMARY KEY and UNIQUE constraints!

-- 3. Set constraints to DEFERRED (checked at commit, not per-row)
SET CONSTRAINTS ALL DEFERRED;

-- 4. Bulk load data
COPY orders FROM '/data/orders.csv' WITH (FORMAT csv, HEADER true);
-- Or use multi-row INSERTs

-- 5. Recreate indexes
CREATE INDEX idx_orders_date ON orders(order_date);
CREATE INDEX idx_orders_user ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

-- 6. Re-enable triggers
ALTER TABLE orders ENABLE TRIGGER ALL;

-- 7. Validate constraints
-- This checks all deferred constraints
COMMIT;

-- 8. Update statistics
ANALYZE orders;
```

**Creating Indexes After Bulk Load:**

```sql
-- ✅ Fast: Build index after loading data
BEGIN;
    -- Load data without indexes (except PK)
    COPY orders FROM '/data/orders.csv' WITH (FORMAT csv);
    
    -- Build indexes on loaded data
    CREATE INDEX CONCURRENTLY idx_orders_date ON orders(order_date);
    CREATE INDEX CONCURRENTLY idx_orders_user ON orders(user_id);
COMMIT;

ANALYZE orders;

-- CONCURRENTLY allows reads/writes during index creation
-- But cannot be used inside a transaction block
```

---

**Method 5: Parallel Bulk Operations**

Use multiple connections for parallel inserts (for very large datasets).

**Python Example with Multiprocessing:**

```python
import psycopg2
import psycopg2.extras
from multiprocessing import Pool, cpu_count

def insert_chunk(args):
    """Insert a chunk of data in a separate process"""
    chunk_data, chunk_id = args
    
    conn = psycopg2.connect("dbname=mydb user=postgres")
    cur = conn.cursor()
    
    try:
        psycopg2.extras.execute_values(
            cur,
            "INSERT INTO users (user_id, name, email) VALUES %s",
            chunk_data,
            page_size=1000
        )
        conn.commit()
        print(f"Chunk {chunk_id} completed: {len(chunk_data)} rows")
        return True
    except Exception as e:
        conn.rollback()
        print(f"Chunk {chunk_id} failed: {e}")
        return False
    finally:
        cur.close()
        conn.close()

def parallel_bulk_insert(data, num_workers=None):
    """
    Split data into chunks and insert in parallel
    """
    if num_workers is None:
        num_workers = cpu_count()
    
    # Split data into chunks
    chunk_size = len(data) // num_workers
    chunks = [
        (data[i:i + chunk_size], i // chunk_size)
        for i in range(0, len(data), chunk_size)
    ]
    
    # Process chunks in parallel
    with Pool(processes=num_workers) as pool:
        results = pool.map(insert_chunk, chunks)
    
    success_count = sum(results)
    print(f"Completed: {success_count}/{len(chunks)} chunks succeeded")

# Usage
data = [(i, f'User{i}', f'user{i}@example.com') for i in range(10000000)]
parallel_bulk_insert(data, num_workers=8)
# Time: ~15-20 seconds for 10M rows (8 workers)
```

**Important Considerations for Parallel Inserts:**

```sql
-- 1. Use UNLOGGED tables for maximum speed (no WAL)
CREATE UNLOGGED TABLE temp_users (
    user_id INT,
    name TEXT,
    email TEXT
);
-- Insert data (extremely fast)
-- Then convert to logged table
ALTER TABLE temp_users SET LOGGED;

-- 2. Partition the table for true parallel inserts
CREATE TABLE orders (
    order_id BIGSERIAL,
    order_date DATE,
    user_id INT
) PARTITION BY RANGE (order_date);

-- Each worker inserts into different partition
-- No lock contention!
```

---

**Method 6: Efficient UPDATES**

Updating millions of rows requires different strategies than inserts.

**Batch Updates:**

```sql
-- ❌ Slow: Update all rows at once (locks entire table)
UPDATE orders
SET status = 'archived'
WHERE order_date < '2020-01-01';
-- Locks millions of rows, blocks other queries

-- ✅ Fast: Batch updates with periodic commits
DO $$
DECLARE
    batch_size INT := 10000;
    rows_updated INT;
BEGIN
    LOOP
        UPDATE orders
        SET status = 'archived'
        WHERE order_id IN (
            SELECT order_id 
            FROM orders
            WHERE order_date < '2020-01-01'
              AND status != 'archived'
            LIMIT batch_size
        );
        
        GET DIAGNOSTICS rows_updated = ROW_COUNT;
        
        -- Exit when no more rows to update
        EXIT WHEN rows_updated = 0;
        
        -- Commit batch
        COMMIT;
        
        -- Log progress
        RAISE NOTICE 'Updated % rows', rows_updated;
        
        -- Small delay to allow other queries to run
        PERFORM pg_sleep(0.1);
    END LOOP;
END $$;
```

**Python Example for Batch Updates:**

```python
import psycopg2

def batch_update(conn, batch_size=10000):
    """
    Update rows in batches to avoid long locks
    """
    cur = conn.cursor()
    total_updated = 0
    
    while True:
        cur.execute("""
            UPDATE orders
            SET status = 'archived'
            WHERE order_id IN (
                SELECT order_id
                FROM orders
                WHERE order_date < '2020-01-01'
                  AND status != 'archived'
                LIMIT %s
            )
        """, (batch_size,))
        
        rows_updated = cur.rowcount
        
        if rows_updated == 0:
            break
        
        conn.commit()
        total_updated += rows_updated
        
        print(f"Updated {total_updated} rows so far...")
        
        # Brief pause
        time.sleep(0.1)
    
    print(f"Total updated: {total_updated} rows")
    cur.close()

# Usage
conn = psycopg2.connect("dbname=mydb user=postgres")
batch_update(conn, batch_size=10000)
conn.close()
```

**UPDATE with JOIN (more efficient than subquery):**

```sql
-- ❌ Slower: Correlated subquery
UPDATE orders o
SET discount_amount = (
    SELECT c.discount_rate * o.total_amount
    FROM customers c
    WHERE c.customer_id = o.customer_id
)
WHERE EXISTS (
    SELECT 1 FROM customers c 
    WHERE c.customer_id = o.customer_id 
      AND c.discount_rate > 0
);

-- ✅ Faster: JOIN-based update
UPDATE orders o
SET discount_amount = c.discount_rate * o.total_amount
FROM customers c
WHERE o.customer_id = c.customer_id
  AND c.discount_rate > 0;
```

**Bulk UPDATE from Staging Table:**

```sql
-- Scenario: Update 5 million orders based on external data

-- Step 1: Load new data into temp table
CREATE TEMP TABLE orders_update (
    order_id BIGINT,
    new_status TEXT,
    new_total DECIMAL(10,2)
);

COPY orders_update FROM '/data/updates.csv' WITH (FORMAT csv);

-- Step 2: Create index on temp table
CREATE INDEX idx_orders_update_id ON orders_update(order_id);

-- Step 3: Batch update
DO $$
DECLARE
    batch_size INT := 50000;
    min_id BIGINT := 0;
    max_id BIGINT;
BEGIN
    SELECT MAX(order_id) INTO max_id FROM orders_update;
    
    WHILE min_id < max_id LOOP
        UPDATE orders o
        SET 
            status = ou.new_status,
            total_amount = ou.new_total,
            updated_at = NOW()
        FROM orders_update ou
        WHERE o.order_id = ou.order_id
          AND o.order_id > min_id
          AND o.order_id <= min_id + batch_size;
        
        min_id := min_id + batch_size;
        COMMIT;
        
        RAISE NOTICE 'Updated orders up to ID %', min_id;
    END LOOP;
END $$;

DROP TABLE orders_update;
```

---

**Method 7: UPSERT (INSERT ... ON CONFLICT)**

Efficiently handle insert-or-update scenarios.

```sql
-- Scenario: Daily data load - insert new, update existing

-- ❌ Slow: Check then insert/update
-- Requires 2 queries per row
DO $$
BEGIN
    IF EXISTS (SELECT 1 FROM users WHERE user_id = 123) THEN
        UPDATE users SET name = 'New Name' WHERE user_id = 123;
    ELSE
        INSERT INTO users VALUES (123, 'New Name', 'email@example.com');
    END IF;
END $$;

-- ✅ Fast: UPSERT in one statement
INSERT INTO users (user_id, name, email)
VALUES (123, 'New Name', 'email@example.com')
ON CONFLICT (user_id)
DO UPDATE SET
    name = EXCLUDED.name,
    email = EXCLUDED.email,
    updated_at = NOW();
```

**Bulk UPSERT Example:**

```python
import psycopg2
import psycopg2.extras

def bulk_upsert(data):
    """
    Efficiently upsert millions of rows
    """
    conn = psycopg2.connect("dbname=mydb user=postgres")
    cur = conn.cursor()
    
    # Prepare the query with multiple rows
    query = """
        INSERT INTO users (user_id, name, email, login_count)
        VALUES %s
        ON CONFLICT (user_id)
        DO UPDATE SET
            name = EXCLUDED.name,
            email = EXCLUDED.email,
            login_count = users.login_count + EXCLUDED.login_count,
            updated_at = NOW()
    """
    
    # Execute in batches
    batch_size = 10000
    for i in range(0, len(data), batch_size):
        batch = data[i:i + batch_size]
        
        psycopg2.extras.execute_values(
            cur,
            query,
            batch,
            page_size=1000
        )
        
        conn.commit()
        print(f"Upserted {min(i + batch_size, len(data))}/{len(data)} rows")
    
    cur.close()
    conn.close()

# Usage: Mix of new and existing users
data = [
    (1, 'User1', 'user1@example.com', 5),      # Existing - will update
    (2, 'User2', 'user2@example.com', 3),      # Existing - will update
    (1000001, 'NewUser', 'new@example.com', 1) # New - will insert
    # ... millions more
]
bulk_upsert(data)
```

---

**Method 8: Optimize Configuration for Bulk Operations**

Temporarily adjust PostgreSQL settings for better bulk performance.

```sql
-- Session-level settings for bulk load

-- 1. Increase work_mem for sorting/hashing
SET work_mem = '256MB';

-- 2. Increase maintenance_work_mem for index creation
SET maintenance_work_mem = '2GB';

-- 3. Disable synchronous_commit (faster, slight risk of data loss)
SET synchronous_commit = OFF;
-- Data is still durable, just slightly delayed

-- 4. Reduce checkpoint frequency during bulk load
-- (These are system-level, not session)
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
SELECT pg_reload_conf();

-- 5. Disable autovacuum for the table during bulk load
ALTER TABLE orders SET (autovacuum_enabled = false);

-- Perform bulk operations here
COPY orders FROM '/data/orders.csv';

-- 6. Re-enable and run manual VACUUM ANALYZE
ALTER TABLE orders SET (autovacuum_enabled = true);
VACUUM ANALYZE orders;

-- 7. Reset settings
RESET work_mem;
RESET maintenance_work_mem;
RESET synchronous_commit;
```

**Full Optimization Script:**

```sql
-- Complete bulk load optimization

BEGIN;

-- Prepare table
ALTER TABLE orders DISABLE TRIGGER ALL;
ALTER TABLE orders SET (autovacuum_enabled = false);

-- Drop non-critical indexes
DROP INDEX IF EXISTS idx_orders_date;
DROP INDEX IF EXISTS idx_orders_status;
-- Keep PRIMARY KEY and FOREIGN KEYs

-- Optimize session
SET work_mem = '256MB';
SET maintenance_work_mem = '2GB';
SET synchronous_commit = OFF;

-- Load data
COPY orders FROM '/data/orders.csv' WITH (FORMAT csv, HEADER true);
-- Result: 10M rows in ~30 seconds

-- Recreate indexes (can't use CONCURRENTLY in transaction)
COMMIT;

CREATE INDEX CONCURRENTLY idx_orders_date ON orders(order_date);
CREATE INDEX CONCURRENTLY idx_orders_status ON orders(status);

-- Cleanup
ALTER TABLE orders ENABLE TRIGGER ALL;
ALTER TABLE orders SET (autovacuum_enabled = true);

-- Update statistics
ANALYZE orders;

-- Reset session settings
RESET ALL;
```

---

**Real-World Example: Loading 50M Rows**

```sql
-- Scenario: Daily ETL loading 50 million orders from data warehouse

-- ❌ Original approach: 2 hours
-- Individual INSERTs with auto-commit

-- ✅ Optimized approach: 5 minutes

-- Step 1: Create staging table (UNLOGGED for speed)
CREATE UNLOGGED TABLE orders_staging (LIKE orders INCLUDING ALL);

-- Step 2: Bulk load into staging (no indexes, no constraints)
COPY orders_staging 
FROM '/data/orders_2024_01_15.csv'
WITH (FORMAT csv, HEADER true);
-- Time: ~2 minutes for 50M rows

-- Step 3: Validate data
SELECT COUNT(*) FROM orders_staging WHERE order_id IS NULL;  -- Should be 0
SELECT COUNT(*) FROM orders_staging WHERE user_id IS NULL;   -- Should be 0

-- Step 4: Move to production table (with deduplication)
INSERT INTO orders
SELECT DISTINCT ON (order_id) *
FROM orders_staging
ON CONFLICT (order_id) DO NOTHING;
-- Time: ~2 minutes

-- Step 5: Update indexes and statistics
ANALYZE orders;
-- Time: ~1 minute

-- Step 6: Cleanup
DROP TABLE orders_staging;

-- Total time: ~5 minutes vs 2 hours (24x faster!)
```

---

**Comparison Table:**

| Method | Speed (1M rows) | Pros | Cons | Best For |
|--------|----------------|------|------|----------|
| **COPY** | 2-5 sec | Fastest, lowest overhead | Requires file/pipe access | Initial loads, migrations |
| **Multi-row INSERT** | 8-12 sec | Fast, flexible | More parsing overhead | Application inserts |
| **execute_values** | 8-12 sec | Easy from code, fast | Requires library support | Python/app inserts |
| **execute_batch** | 10-15 sec | Good balance | Slightly slower | General purpose |
| **Batched singles** | 30 sec | Resumable on failure | Slower | Long-running jobs |
| **Individual + txn** | 2-3 min | Simple code | Slow | Small datasets only |
| **Auto-commit singles** | 30+ min | None! | Extremely slow | ❌ Never use |

---

**Performance Tips Summary:**

**For INSERTS:**
1. Use `COPY` for file-based loads (fastest)
2. Use multi-row INSERTs or `execute_values` for programmatic inserts
3. Always use transactions (avoid auto-commit)
4. Drop/disable indexes during bulk load, recreate after
5. Disable triggers temporarily
6. Consider UNLOGGED tables for staging
7. Use parallel workers for very large datasets (10M+ rows)
8. Increase `work_mem` and `maintenance_work_mem`
9. Run `ANALYZE` after bulk operations

**For UPDATES:**
1. Batch updates (10K-50K rows at a time)
2. Use JOIN instead of correlated subqueries
3. Update from staging table for large datasets
4. Use `ON CONFLICT` for upsert scenarios
5. Create indexes on update criteria
6. Consider partitioning for very large tables

**For BOTH:**
1. Monitor with `pg_stat_progress_copy` and `pg_stat_activity`
2. Test batch sizes (typically 1K-10K optimal)
3. Use connection pooling (PgBouncer)
4. Schedule during low-traffic periods
5. Keep transaction logs from growing too large

---

**Monitoring Bulk Operations:**

```sql
-- Monitor COPY progress (PostgreSQL 14+)
SELECT 
    pid,
    relid::regclass,
    command,
    type,
    bytes_processed,
    bytes_total,
    tuples_processed,
    tuples_excluded
FROM pg_stat_progress_copy;

-- Monitor active queries
SELECT 
    pid,
    usename,
    state,
    query,
    query_start,
    NOW() - query_start as duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY query_start;

-- Check table bloat after bulk operations
SELECT
    schemaname,
    tablename,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size,
    pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename) - 
                   pg_relation_size(schemaname||'.'||tablename)) AS index_size
FROM pg_tables
WHERE tablename = 'orders';

-- Monitor locks
SELECT 
    pid,
    mode,
    granted,
    relation::regclass
FROM pg_locks
WHERE NOT granted;
```

---

**Key Takeaways:**

1. **COPY is king** - Use it whenever possible for file-based loads
2. **Batch everything** - Never do row-by-row operations at scale
3. **Transactions matter** - Wrap batches in transactions to reduce commit overhead
4. **Indexes are expensive** - Drop before bulk load, recreate after
5. **Use staging tables** - Validate data before moving to production
6. **Parallel processing** - Use multiple workers for 10M+ rows
7. **Monitor and measure** - Always benchmark different approaches
8. **Balance speed vs. safety** - Consider trade-offs (e.g., `synchronous_commit`)

---


**3. PostgreSQL vs TimescaleDB for time-series data.**

TimescaleDB is a PostgreSQL extension that adds time-series database capabilities to PostgreSQL. It's not a replacement for PostgreSQL, but rather an enhancement that transforms PostgreSQL into a powerful time-series database while maintaining full SQL compatibility and PostgreSQL's reliability.

---

**What is TimescaleDB?**

TimescaleDB is an open-source extension that:
- Builds on top of PostgreSQL (uses PostgreSQL's storage engine, replication, etc.)
- Adds automatic partitioning by time (hypertables)
- Optimizes queries and storage for time-series workloads
- Provides specialized functions for time-series analysis
- Maintains 100% SQL compatibility (you can use all PostgreSQL features)

---

**Key Differences:**

| Aspect | PostgreSQL | TimescaleDB |
|--------|------------|-------------|
| **Architecture** | Relational database | PostgreSQL + time-series extension |
| **Partitioning** | Manual (declarative partitioning) | Automatic (hypertables) |
| **Data Compression** | Limited (table-level) | Advanced (chunk-level, columnar) |
| **Time-Series Functions** | Basic (date/time functions) | Specialized (time_bucket, interpolate, etc.) |
| **Continuous Aggregates** | Manual (materialized views) | Automatic refresh (continuous aggregates) |
| **Data Retention** | Manual cleanup | Automatic data retention policies |
| **Query Performance** | Good for small-medium TS data | Optimized for large-scale TS data |
| **Write Performance** | Moderate | Higher (optimized inserts) |
| **Storage Efficiency** | Standard | Better (compression) |

---

**When to Use PostgreSQL:**

**✅ Use PostgreSQL when:**
1. **Small to medium time-series datasets** (< 100M rows per table)
2. **Mixed workloads** - need both relational and time-series data
3. **Complex relational queries** - frequent JOINs with non-time-series tables
4. **PostgreSQL-specific features** - need extensions TimescaleDB doesn't support
5. **Budget constraints** - don't want to add another tool/extension
6. **Simple time-series needs** - basic time filtering and aggregation
7. **Data model flexibility** - frequently changing schema

**Example Use Case:**
```sql
-- E-commerce: Orders table with timestamps
-- Mostly relational queries, occasional time-based analytics
CREATE TABLE orders (
    order_id BIGSERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    order_date TIMESTAMP NOT NULL,
    total_amount DECIMAL(10,2),
    status TEXT
);

-- Mostly query like this:
SELECT u.name, o.order_date, o.total_amount
FROM orders o
JOIN users u ON o.user_id = u.user_id
WHERE o.order_date >= '2024-01-01';

-- Occasional time-series query:
SELECT DATE_TRUNC('day', order_date) as day, COUNT(*) as orders
FROM orders
WHERE order_date >= '2024-01-01'
GROUP BY day;
-- PostgreSQL handles this fine!
```

---

**When to Use TimescaleDB:**

**✅ Use TimescaleDB when:**
1. **Large-scale time-series data** (> 100M rows, terabytes of data)
2. **High ingestion rates** - millions of data points per second
3. **Time-based queries are primary** - most queries filter by time
4. **Need compression** - reduce storage costs significantly
5. **Continuous aggregates** - pre-compute rollups (minute/hour/day summaries)
6. **Data retention policies** - automatically delete old data
7. **IoT/Sensor data** - temperature, metrics, logs, events
8. **Real-time analytics** - dashboards with time-series visualizations

**Example Use Case:**
```sql
-- IoT sensor data: 10,000 sensors × 1 reading/minute = 14.4M rows/day
-- After 1 year: ~5.2 billion rows

-- Convert regular table to hypertable
CREATE TABLE sensor_readings (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INT NOT NULL,
    temperature DECIMAL(5,2),
    humidity DECIMAL(5,2),
    pressure DECIMAL(8,2)
);

-- Create hypertable (automatic partitioning by time)
SELECT create_hypertable('sensor_readings', 'time', 
    chunk_time_interval => INTERVAL '1 day');

-- Insert millions of rows (optimized)
INSERT INTO sensor_readings VALUES
    ('2024-01-01 00:00:00', 1, 22.5, 45.0, 1013.25),
    ('2024-01-01 00:01:00', 1, 22.6, 45.1, 1013.26),
    -- ... millions more

-- Time-series optimized queries
SELECT 
    time_bucket('1 hour', time) as hour,
    sensor_id,
    AVG(temperature) as avg_temp,
    MAX(temperature) as max_temp,
    MIN(temperature) as min_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '7 days'
  AND sensor_id = 1
GROUP BY hour, sensor_id
ORDER BY hour;
-- Much faster than PostgreSQL on same data!
```

---

**Performance Comparison:**

**Write Performance:**

```sql
-- PostgreSQL: Inserting 10M rows
BEGIN;
INSERT INTO metrics (time, device_id, value) VALUES
    ('2024-01-01 00:00:00', 1, 10.5),
    ('2024-01-01 00:00:01', 1, 10.6),
    -- ... 10M rows
COMMIT;
-- Time: ~2-3 minutes (with indexes)

-- TimescaleDB: Same 10M rows
BEGIN;
INSERT INTO metrics (time, device_id, value) VALUES
    ('2024-01-01 00:00:00', 1, 10.5),
    ('2024-01-01 00:00:01', 1, 10.6),
    -- ... 10M rows
COMMIT;
-- Time: ~30-60 seconds (hypertable optimization)
-- 2-4x faster!
```

**Query Performance:**

```sql
-- Query: Average temperature per hour for last 30 days
-- Table: 1 billion rows, 10,000 sensors

-- PostgreSQL approach:
SELECT 
    DATE_TRUNC('hour', time) as hour,
    AVG(temperature) as avg_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '30 days'
GROUP BY hour
ORDER BY hour;
-- Time: ~45-60 seconds (full table scan or large index scan)

-- TimescaleDB approach:
SELECT 
    time_bucket('1 hour', time) as hour,
    AVG(temperature) as avg_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '30 days'
GROUP BY hour
ORDER BY hour;
-- Time: ~5-10 seconds (chunk exclusion + optimized functions)
-- 5-10x faster!

-- With continuous aggregate (pre-computed):
SELECT hour, avg_temp
FROM sensor_readings_hourly  -- Materialized view
WHERE hour >= NOW() - INTERVAL '30 days'
ORDER BY hour;
-- Time: ~0.1 seconds (reads pre-aggregated data)
-- 500x faster!
```

**Storage Comparison:**

```sql
-- Scenario: 1 billion sensor readings

-- PostgreSQL storage:
-- Raw data: ~100 GB
-- Indexes: ~30 GB
-- Total: ~130 GB

-- TimescaleDB storage (with compression):
-- Raw data: ~100 GB
-- Compressed: ~10 GB (10x compression!)
-- Indexes: ~5 GB (smaller with compression)
-- Total: ~15 GB

-- Storage savings: ~88% reduction!
```

---

**TimescaleDB Key Features:**

**1. Hypertables (Automatic Partitioning):**

```sql
-- Create regular table
CREATE TABLE metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id INT NOT NULL,
    value DOUBLE PRECISION
);

-- Convert to hypertable (automatic partitioning)
SELECT create_hypertable('metrics', 'time', 
    chunk_time_interval => INTERVAL '1 day');

-- TimescaleDB automatically:
-- - Creates chunks (partitions) by time
-- - Manages chunk lifecycle
-- - Optimizes queries with chunk exclusion
-- - Handles data movement between chunks

-- View chunks
SELECT * FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics';
```

**2. Compression:**

```sql
-- Enable compression on hypertable
ALTER TABLE metrics SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'device_id',
    timescaledb.compress_orderby = 'time DESC'
);

-- Compress old data (e.g., older than 7 days)
SELECT compress_chunk(chunk)
FROM timescaledb_information.chunks
WHERE hypertable_name = 'metrics'
  AND range_end < NOW() - INTERVAL '7 days';

-- Compression ratio: Often 10-20x reduction!

-- Query compressed data (automatically decompressed)
SELECT * FROM metrics
WHERE time >= NOW() - INTERVAL '30 days';
-- Works transparently!
```

**3. Continuous Aggregates:**

```sql
-- Create continuous aggregate (automatically refreshed)
CREATE MATERIALIZED VIEW metrics_hourly
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) as hour,
    device_id,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value,
    COUNT(*) as count
FROM metrics
GROUP BY hour, device_id;

-- Set refresh policy (refresh every hour)
SELECT add_continuous_aggregate_policy('metrics_hourly',
    start_offset => INTERVAL '3 hours',
    end_offset => INTERVAL '1 hour',
    schedule_interval => INTERVAL '1 hour');

-- Query the aggregate (fast!)
SELECT * FROM metrics_hourly
WHERE hour >= NOW() - INTERVAL '7 days'
  AND device_id = 1
ORDER BY hour;

-- TimescaleDB automatically:
-- - Refreshes data incrementally
-- - Only processes new data (not entire table)
-- - Handles overlapping time ranges correctly
```

**4. Data Retention Policies:**

```sql
-- Automatically delete data older than 90 days
SELECT add_retention_policy('metrics', INTERVAL '90 days');

-- Or more complex: Keep raw data for 7 days, hourly aggregates for 1 year
SELECT add_retention_policy('metrics', INTERVAL '7 days');
-- Keep hourly aggregates longer
SELECT add_retention_policy('metrics_hourly', INTERVAL '1 year');

-- Policies run automatically (via background jobs)
-- No manual cleanup needed!
```

**5. Time-Series Functions:**

```sql
-- time_bucket: Group by time intervals
SELECT 
    time_bucket('5 minutes', time) as bucket,
    AVG(temperature) as avg_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '1 day'
GROUP BY bucket;

-- time_bucket_gapfill: Fill gaps in time-series
SELECT 
    time_bucket_gapfill('1 hour', time, 
        start => NOW() - INTERVAL '7 days',
        finish => NOW()) as hour,
    AVG(temperature) as avg_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '7 days'
GROUP BY hour;
-- Returns NULL for missing hours (or use LOCF interpolation)

-- interpolate: Linear interpolation
SELECT 
    time_bucket('1 hour', time) as hour,
    interpolate(AVG(temperature)) as avg_temp
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '7 days'
GROUP BY hour;
-- Fills gaps with interpolated values

-- first() and last(): First/last value in group
SELECT 
    time_bucket('1 day', time) as day,
    first(temperature, time) as first_temp,
    last(temperature, time) as last_temp
FROM sensor_readings
GROUP BY day;

-- histogram: Distribution analysis
SELECT 
    time_bucket('1 hour', time) as hour,
    histogram(temperature, 0, 100, 10) as temp_distribution
FROM sensor_readings
WHERE time >= NOW() - INTERVAL '1 day'
GROUP BY hour;
```

**6. Downsampling:**

```sql
-- Example: Keep raw data for 7 days, hourly aggregates forever

-- Raw data hypertable
CREATE TABLE raw_metrics (...);
SELECT create_hypertable('raw_metrics', 'time');

-- Hourly aggregate hypertable
CREATE TABLE metrics_hourly (...);
SELECT create_hypertable('metrics_hourly', 'time');

-- Continuous aggregate that downsamples
CREATE MATERIALIZED VIEW metrics_hourly_agg
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 hour', time) as hour,
    device_id,
    AVG(value) as avg_value
FROM raw_metrics
GROUP BY hour, device_id;

-- Retention: Delete raw data after 7 days
SELECT add_retention_policy('raw_metrics', INTERVAL '7 days');

-- Keep hourly aggregates forever (or longer retention)
-- Result: Storage optimized, queries still fast!
```

---

**Migration from PostgreSQL to TimescaleDB:**

```sql
-- Step 1: Install TimescaleDB extension
CREATE EXTENSION IF NOT EXISTS timescaledb;

-- Step 2: Convert existing table to hypertable
-- (Only if table is empty or you're okay with downtime)
SELECT create_hypertable('existing_table', 'time_column');

-- OR: Migrate data in batches

-- Step 2a: Create new hypertable
CREATE TABLE metrics_new (LIKE metrics INCLUDING ALL);
SELECT create_hypertable('metrics_new', 'time');

-- Step 2b: Migrate data in batches
INSERT INTO metrics_new
SELECT * FROM metrics
WHERE time >= '2024-01-01' AND time < '2024-02-01';

-- Step 2c: Swap tables
BEGIN;
    ALTER TABLE metrics RENAME TO metrics_old;
    ALTER TABLE metrics_new RENAME TO metrics;
COMMIT;

-- Step 3: Drop old table (after verification)
DROP TABLE metrics_old;
```

---

**Real-World Scenarios:**

**Scenario 1: IoT Sensor Monitoring**

```sql
-- Requirements:
-- - 50,000 sensors
-- - 1 reading per sensor per minute
-- - Store 2 years of data
-- - Real-time dashboards
-- - Alert on anomalies

-- PostgreSQL approach:
CREATE TABLE sensor_data (
    time TIMESTAMP NOT NULL,
    sensor_id INT NOT NULL,
    value DOUBLE PRECISION,
    PRIMARY KEY (time, sensor_id)
);
CREATE INDEX idx_sensor_data_sensor_time ON sensor_data(sensor_id, time);

-- Challenges:
-- - Table grows to ~52 billion rows
-- - Queries slow down significantly
-- - Storage costs high
-- - Maintenance (VACUUM, REINDEX) takes hours

-- TimescaleDB approach:
CREATE TABLE sensor_data (
    time TIMESTAMPTZ NOT NULL,
    sensor_id INT NOT NULL,
    value DOUBLE PRECISION
);
SELECT create_hypertable('sensor_data', 'time', 
    chunk_time_interval => INTERVAL '1 day');

-- Add compression
ALTER TABLE sensor_data SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'sensor_id'
);

-- Create continuous aggregates for dashboards
CREATE MATERIALIZED VIEW sensor_data_minute
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 minute', time) as minute,
    sensor_id,
    AVG(value) as avg_value,
    MAX(value) as max_value,
    MIN(value) as min_value
FROM sensor_data
GROUP BY minute, sensor_id;

-- Result:
-- - Faster queries (chunk exclusion)
-- - 10x storage reduction (compression)
-- - Fast dashboards (continuous aggregates)
-- - Automatic retention (delete old raw data)
```

**Scenario 2: Application Metrics**

```sql
-- Requirements:
-- - Application performance metrics
-- - User behavior events
-- - Mix of relational and time-series queries
-- - Need JOINs with users, products tables

-- Choice: PostgreSQL (not TimescaleDB)
-- Reason: Frequent JOINs with relational tables

CREATE TABLE events (
    event_id BIGSERIAL PRIMARY KEY,
    user_id INT REFERENCES users(user_id),
    product_id INT REFERENCES products(product_id),
    event_type TEXT,
    event_time TIMESTAMP NOT NULL,
    properties JSONB
);

CREATE INDEX idx_events_time ON events(event_time);
CREATE INDEX idx_events_user_time ON events(user_id, event_time);

-- Queries mix time-series and relational:
SELECT 
    u.name,
    DATE_TRUNC('day', e.event_time) as day,
    COUNT(*) as event_count
FROM events e
JOIN users u ON e.user_id = u.user_id
WHERE e.event_time >= '2024-01-01'
GROUP BY u.name, day;

-- PostgreSQL handles this well because:
-- - Frequent JOINs with relational tables
-- - Schema flexibility (adding event types)
-- - Moderate data volume (< 1B rows)
```

**Scenario 3: Financial Market Data**

```sql
-- Requirements:
-- - Stock prices: 10,000 symbols × 1 price/second = 864M rows/day
-- - Store 5 years of data = ~1.5 trillion rows
-- - Real-time queries for trading
-- - Historical analysis

-- Choice: TimescaleDB
-- Reason: Massive scale, time-series focused

CREATE TABLE stock_prices (
    time TIMESTAMPTZ NOT NULL,
    symbol TEXT NOT NULL,
    price DECIMAL(10,2),
    volume BIGINT
);

SELECT create_hypertable('stock_prices', 'time',
    chunk_time_interval => INTERVAL '1 day',
    partitioning_column => 'symbol',
    number_partitions => 10);

-- Compression (compress data older than 1 day)
ALTER TABLE stock_prices SET (
    timescaledb.compress,
    timescaledb.compress_segmentby = 'symbol',
    timescaledb.compress_orderby = 'time DESC'
);

-- Continuous aggregate for candlestick charts
CREATE MATERIALIZED VIEW stock_prices_1min
WITH (timescaledb.continuous) AS
SELECT 
    time_bucket('1 minute', time) as minute,
    symbol,
    first(price, time) as open,
    MAX(price) as high,
    MIN(price) as low,
    last(price, time) as close,
    SUM(volume) as volume
FROM stock_prices
GROUP BY minute, symbol;

-- Query recent data (raw, fast)
SELECT * FROM stock_prices
WHERE symbol = 'AAPL'
  AND time >= NOW() - INTERVAL '1 hour'
ORDER BY time;

-- Query historical data (aggregated, very fast)
SELECT * FROM stock_prices_1min
WHERE symbol = 'AAPL'
  AND minute >= NOW() - INTERVAL '30 days'
ORDER BY minute;
```

---

**Limitations and Considerations:**

**TimescaleDB Limitations:**

1. **Extension dependency** - Must be installed and maintained
2. **Learning curve** - New concepts (hypertables, chunks, policies)
3. **Partitioning flexibility** - Automatic partitioning by time (less flexible than manual)
4. **Some PostgreSQL features** - Some extensions may not work perfectly
5. **Chunk management** - Need to understand chunk lifecycle

**PostgreSQL Limitations for Time-Series:**

1. **Manual partitioning** - Must manage partitions yourself
2. **No automatic compression** - Manual compression solutions needed
3. **Query performance** - Slower on very large time-series datasets
4. **No continuous aggregates** - Must manually refresh materialized views
5. **Storage costs** - Higher storage requirements

---

**Decision Matrix:**

**Choose PostgreSQL if:**
- ✅ Data volume < 100M rows per time-series table
- ✅ Mixed relational + time-series queries
- ✅ Frequent JOINs with non-time-series tables
- ✅ Schema changes are common
- ✅ Budget/tooling constraints

**Choose TimescaleDB if:**
- ✅ Data volume > 100M rows (or growing rapidly)
- ✅ Primary queries are time-series focused
- ✅ Need compression (storage cost reduction)
- ✅ Want automatic data retention
- ✅ Need continuous aggregates for dashboards
- ✅ High ingestion rates (millions of inserts/second)

**Hybrid Approach:**
- Use PostgreSQL for relational data
- Use TimescaleDB for time-series data
- JOIN them when needed (both are PostgreSQL!)

```sql
-- Example: Relational data in PostgreSQL
CREATE TABLE devices (
    device_id INT PRIMARY KEY,
    device_name TEXT,
    location TEXT
);

-- Time-series data in TimescaleDB
CREATE TABLE device_metrics (
    time TIMESTAMPTZ NOT NULL,
    device_id INT NOT NULL,
    temperature DOUBLE PRECISION
);
SELECT create_hypertable('device_metrics', 'time');

-- JOIN them when needed
SELECT 
    d.device_name,
    d.location,
    dm.time,
    dm.temperature
FROM device_metrics dm
JOIN devices d ON dm.device_id = d.device_id
WHERE dm.time >= NOW() - INTERVAL '1 day'
ORDER BY dm.time;
```

---

**Key Takeaways:**

1. **TimescaleDB is PostgreSQL** - It's an extension, not a separate database
2. **Use TimescaleDB for large-scale time-series** - Better performance and features
3. **Use PostgreSQL for small-medium or mixed workloads** - Simpler, no extra dependencies
4. **Hypertables provide automatic partitioning** - No manual partition management
5. **Compression saves storage** - Often 10-20x reduction
6. **Continuous aggregates accelerate dashboards** - Pre-computed rollups
7. **You can use both** - Different tables can use different approaches
8. **Migration is straightforward** - Convert tables to hypertables incrementally
9. **SQL compatibility** - All PostgreSQL SQL works with TimescaleDB
10. **Performance scales** - TimescaleDB handles billions of rows efficiently

---


**4. Connection pooling: Why and how? (PgBouncer)**

Answer:


**5. Sharding vs Replication vs Partitioning.**

These are three different database scaling strategies that solve different problems. Understanding when to use each is crucial for building scalable systems.

---

**Quick Comparison:**

| Aspect | Partitioning | Replication | Sharding |
|--------|--------------|-------------|----------|
| **Purpose** | Query performance | High availability, read scaling | Write scaling, horizontal scaling |
| **Data Distribution** | Single database, multiple tables/files | Multiple databases, same data | Multiple databases, different data |
| **Complexity** | Low | Medium | High |
| **Scalability** | Vertical (single server) | Horizontal (read-only) | Horizontal (read + write) |
| **Failure Impact** | Entire database down | Minimal (failover to replica) | One shard down, others work |
| **Query Complexity** | Simple | Simple | Complex (cross-shard queries) |
| **Use Case** | Large tables, time-series data | High availability, read-heavy | Massive datasets, write-heavy |

---

## 1. Partitioning

**What is Partitioning?**

Partitioning splits a large table into smaller physical pieces (partitions) within the same database instance. The table appears as one logical table to applications, but PostgreSQL stores data in multiple physical files.

**Key Characteristics:**
- Single database server
- Improves query performance (scans smaller partitions)
- Improves maintenance (VACUUM, REINDEX on partitions)
- Transparent to applications (looks like one table)
- NOT for scaling writes across multiple servers

---

**Types of Partitioning in PostgreSQL:**

**1. Range Partitioning (most common):**

```sql
-- Partition orders by date
CREATE TABLE orders (
    order_id BIGSERIAL,
    user_id INT,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- Create partitions for each month
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Indexes on each partition
CREATE INDEX idx_orders_2024_01_user ON orders_2024_01(user_id);
CREATE INDEX idx_orders_2024_02_user ON orders_2024_02(user_id);
CREATE INDEX idx_orders_2024_03_user ON orders_2024_03(user_id);

-- Queries automatically use partition pruning
EXPLAIN SELECT * FROM orders
WHERE order_date >= '2024-02-15' AND order_date < '2024-02-20';

/*
Output shows only orders_2024_02 partition is scanned!
Seq Scan on orders_2024_02
  Filter: (order_date >= '2024-02-15' AND order_date < '2024-02-20')
*/
```

**2. List Partitioning:**

```sql
-- Partition by country
CREATE TABLE users (
    user_id INT,
    name TEXT,
    country TEXT NOT NULL,
    created_at TIMESTAMP
) PARTITION BY LIST (country);

CREATE TABLE users_usa PARTITION OF users
    FOR VALUES IN ('USA', 'US');

CREATE TABLE users_india PARTITION OF users
    FOR VALUES IN ('India', 'IN');

CREATE TABLE users_uk PARTITION OF users
    FOR VALUES IN ('UK', 'GB', 'United Kingdom');

CREATE TABLE users_other PARTITION OF users
    DEFAULT;  -- Catch-all for other countries

-- Query only scans relevant partition
SELECT * FROM users WHERE country = 'USA';
-- Only scans users_usa partition
```

**3. Hash Partitioning:**

```sql
-- Partition by hash of user_id (distribute evenly)
CREATE TABLE events (
    event_id BIGSERIAL,
    user_id INT NOT NULL,
    event_type TEXT,
    event_time TIMESTAMP
) PARTITION BY HASH (user_id);

-- Create 4 partitions (0-3)
CREATE TABLE events_0 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE events_1 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE events_2 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE events_3 PARTITION OF events
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);

-- Data distributed evenly across partitions
-- Good for parallel query processing
```

**Benefits of Partitioning:**

```sql
-- 1. Faster queries (partition pruning)
SELECT * FROM orders
WHERE order_date >= '2024-02-01' AND order_date < '2024-03-01';
-- Only scans orders_2024_02 (not entire table)

-- 2. Faster maintenance
VACUUM orders_2024_01;  -- Only vacuum one partition
REINDEX TABLE orders_2024_01;  -- Only reindex one partition

-- 3. Easy archival
-- Detach old partition
ALTER TABLE orders DETACH PARTITION orders_2023_01;
-- Archive to separate tablespace or drop
DROP TABLE orders_2023_01;

-- 4. Better index performance
-- Smaller indexes per partition = faster lookups
-- Can have different indexes per partition

-- 5. Parallel query execution
-- PostgreSQL can scan multiple partitions in parallel
SET max_parallel_workers_per_gather = 4;
SELECT COUNT(*) FROM orders;  -- Scans 4 partitions in parallel
```

**Partitioning Limitations:**

```sql
-- ❌ Still one database server (not distributed)
-- ❌ Doesn't scale writes across multiple machines
-- ❌ Cross-partition queries can be slow
-- ❌ Foreign keys to partitioned tables can be tricky
-- ❌ All partitions must have same columns
```

---

## 2. Replication

**What is Replication?**

Replication creates exact copies of the entire database on multiple servers. One server is the primary (master) that accepts writes, and other servers are replicas (slaves/standby) that receive changes from the primary.

**Key Characteristics:**
- Multiple database servers with same data
- Improves read performance (scale reads across replicas)
- Provides high availability (failover if primary fails)
- Primary handles all writes
- Does NOT scale writes (all writes still go to one server)

---

**Types of Replication in PostgreSQL:**

**1. Streaming Replication (most common):**

```sql
-- PRIMARY SERVER SETUP
-- postgresql.conf
wal_level = replica
max_wal_senders = 5
max_replication_slots = 5
hot_standby = on

-- Create replication user
CREATE USER replicator REPLICATION LOGIN PASSWORD 'securepass';

-- pg_hba.conf (allow replication connections)
host    replication     replicator      replica_ip/32      md5

-- REPLICA SERVER SETUP
-- Use pg_basebackup to create initial copy
pg_basebackup -h primary_server -D /var/lib/postgresql/data -U replicator -P -v -R

-- standby.signal file (marks server as replica)
touch /var/lib/postgresql/data/standby.signal

-- postgresql.conf on replica
hot_standby = on
primary_conninfo = 'host=primary_server port=5432 user=replicator password=securepass'

-- Start replica
pg_ctl start
```

**Connection Routing:**

```python
# Application code with read/write splitting
import psycopg2

# Write connection (to primary)
write_conn = psycopg2.connect(
    host='primary.example.com',
    database='mydb',
    user='appuser'
)

# Read connections (to replicas)
read_conn_1 = psycopg2.connect(
    host='replica1.example.com',
    database='mydb',
    user='appuser'
)

read_conn_2 = psycopg2.connect(
    host='replica2.example.com',
    database='mydb',
    user='appuser'
)

# Write operations go to primary
def create_order(user_id, amount):
    cur = write_conn.cursor()
    cur.execute(
        "INSERT INTO orders (user_id, amount) VALUES (%s, %s)",
        (user_id, amount)
    )
    write_conn.commit()

# Read operations go to replicas (load balanced)
def get_order(order_id):
    import random
    read_conn = random.choice([read_conn_1, read_conn_2])
    
    cur = read_conn.cursor()
    cur.execute("SELECT * FROM orders WHERE order_id = %s", (order_id,))
    return cur.fetchone()
```

**2. Logical Replication:**

```sql
-- Allows selective replication (specific tables, filtered data)

-- PRIMARY SERVER
-- Create publication (what to replicate)
CREATE PUBLICATION mypub FOR TABLE users, orders;

-- Or replicate all tables
CREATE PUBLICATION all_tables FOR ALL TABLES;

-- Or replicate with filter
CREATE PUBLICATION active_users_pub FOR TABLE users WHERE (is_active = true);

-- REPLICA SERVER
-- Create subscription (connect to publication)
CREATE SUBSCRIPTION mysub
    CONNECTION 'host=primary_server dbname=mydb user=replicator password=pass'
    PUBLICATION mypub;

-- Check replication status
SELECT * FROM pg_stat_subscription;
```

**Benefits of Replication:**

```sql
-- 1. High Availability (HA)
-- If primary fails, promote replica to primary
pg_ctl promote

-- 2. Read Scaling
-- Distribute read queries across multiple replicas
-- Example: 1 primary + 5 replicas = 5x read capacity

-- 3. Zero-downtime maintenance
-- Upgrade replicas one by one
-- Perform reads from other replicas during maintenance

-- 4. Geographic distribution
-- Place replicas close to users (low latency reads)
-- Primary in US, replica in EU, replica in Asia

-- 5. Backup without impact
-- Take backups from replica (doesn't affect primary)
pg_dump -h replica_server mydb > backup.sql

-- 6. Analytics without impact
-- Run heavy analytics queries on replica
SELECT /* Run on replica */
    user_id,
    COUNT(*) as order_count,
    SUM(amount) as total_spent
FROM orders
GROUP BY user_id;
-- Doesn't slow down primary!
```

**Replication Lag:**

```sql
-- Check replication lag
-- On replica:
SELECT 
    now() - pg_last_xact_replay_timestamp() AS replication_lag;

-- On primary:
SELECT 
    client_addr,
    state,
    sync_state,
    write_lag,
    flush_lag,
    replay_lag
FROM pg_stat_replication;

-- If replication lag is high (e.g., 10 seconds):
-- - Reads from replica might return stale data
-- - Need to read from primary for critical queries
```

**Handling Replication Lag:**

```python
def get_user_with_consistency(user_id):
    """
    For critical reads, check primary
    For non-critical reads, use replica
    """
    # Critical: Just updated user, need fresh data
    if needs_strong_consistency:
        return read_from_primary(user_id)
    
    # Non-critical: Okay with slightly stale data
    return read_from_replica(user_id)

# Or wait for replication
def update_and_read(user_id, new_name):
    # Update on primary
    update_user(user_id, new_name)
    
    # Wait a bit for replication
    time.sleep(0.5)
    
    # Now read from replica
    return get_user_from_replica(user_id)
```

---

## 3. Sharding

**What is Sharding?**

Sharding splits data across multiple independent database servers (shards). Each shard contains a subset of the data, and together they form the complete dataset. Unlike replication (same data everywhere), sharding distributes different data to different servers.

**Key Characteristics:**
- Multiple database servers with different data
- Scales both reads AND writes horizontally
- Each shard is an independent PostgreSQL instance
- Application must know which shard to query
- Most complex to implement

---

**Sharding Strategies:**

**1. Range-Based Sharding:**

```sql
-- Shard by user_id range
-- Shard 1: user_id 1 to 1,000,000
-- Shard 2: user_id 1,000,001 to 2,000,000
-- Shard 3: user_id 2,000,001 to 3,000,000

-- Application logic determines shard
def get_shard_for_user(user_id):
    if user_id <= 1_000_000:
        return shard1_connection
    elif user_id <= 2_000_000:
        return shard2_connection
    else:
        return shard3_connection

def get_user(user_id):
    shard = get_shard_for_user(user_id)
    cur = shard.cursor()
    cur.execute("SELECT * FROM users WHERE user_id = %s", (user_id,))
    return cur.fetchone()

# On Shard 1
CREATE TABLE users (
    user_id INT PRIMARY KEY,
    name TEXT,
    email TEXT
);
-- Contains users 1 to 1,000,000

# On Shard 2
-- Same schema, different data (users 1,000,001 to 2,000,000)

# On Shard 3
-- Same schema, different data (users 2,000,001 to 3,000,000)
```

**2. Hash-Based Sharding:**

```python
import hashlib

def get_shard_by_hash(user_id, num_shards=4):
    """
    Better data distribution than range-based
    """
    hash_value = int(hashlib.md5(str(user_id).encode()).hexdigest(), 16)
    shard_id = hash_value % num_shards
    return shards[shard_id]

# User 123 -> Shard 2
# User 456 -> Shard 0
# User 789 -> Shard 3
# More evenly distributed!

def create_user(user_id, name, email):
    shard = get_shard_by_hash(user_id)
    cur = shard.cursor()
    cur.execute(
        "INSERT INTO users (user_id, name, email) VALUES (%s, %s, %s)",
        (user_id, name, email)
    )
    shard.commit()
```

**3. Geographic Sharding:**

```python
# Shard by geography
shards = {
    'US': us_shard_connection,
    'EU': eu_shard_connection,
    'ASIA': asia_shard_connection,
}

def get_shard_for_country(country):
    if country in ['USA', 'Canada', 'Mexico']:
        return shards['US']
    elif country in ['UK', 'Germany', 'France']:
        return shards['EU']
    else:
        return shards['ASIA']

def get_users_by_country(country):
    shard = get_shard_for_country(country)
    cur = shard.cursor()
    cur.execute("SELECT * FROM users WHERE country = %s", (country,))
    return cur.fetchall()
```

**4. Entity-Based Sharding:**

```python
# Shard by entity type
# Shard 1: All user-related data for user_id % 4 == 0
# Shard 2: All user-related data for user_id % 4 == 1
# etc.

# Each shard contains:
# - users table
# - orders table (for those users)
# - order_items table (for those orders)

def get_shard(user_id):
    return shards[user_id % 4]

def get_user_orders(user_id):
    shard = get_shard(user_id)
    
    # Can join within shard (same database)
    cur = shard.cursor()
    cur.execute("""
        SELECT u.name, o.order_date, o.total_amount
        FROM users u
        JOIN orders o ON u.user_id = o.user_id
        WHERE u.user_id = %s
    """, (user_id,))
    return cur.fetchall()
```

---

**Implementing Sharding with Foreign Data Wrapper:**

```sql
-- PostgreSQL can use postgres_fdw to query other PostgreSQL servers

-- On coordinator/application server
CREATE EXTENSION postgres_fdw;

-- Add foreign servers (shards)
CREATE SERVER shard1
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'shard1.example.com', port '5432', dbname 'mydb');

CREATE SERVER shard2
    FOREIGN DATA WRAPPER postgres_fdw
    OPTIONS (host 'shard2.example.com', port '5432', dbname 'mydb');

-- Create user mapping
CREATE USER MAPPING FOR myuser
    SERVER shard1
    OPTIONS (user 'remote_user', password 'pass');

CREATE USER MAPPING FOR myuser
    SERVER shard2
    OPTIONS (user 'remote_user', password 'pass');

-- Create foreign tables
CREATE FOREIGN TABLE users_shard1 (
    user_id INT,
    name TEXT,
    email TEXT
)
SERVER shard1
OPTIONS (schema_name 'public', table_name 'users');

CREATE FOREIGN TABLE users_shard2 (
    user_id INT,
    name TEXT,
    email TEXT
)
SERVER shard2
OPTIONS (schema_name 'public', table_name 'users');

-- Create view that unions shards (optional)
CREATE VIEW users_all AS
SELECT * FROM users_shard1
UNION ALL
SELECT * FROM users_shard2;

-- Query all users (across shards)
SELECT * FROM users_all WHERE name LIKE 'John%';
-- PostgreSQL pushes filter to each shard
```

---

**Sharding Challenges:**

**1. Cross-Shard Queries:**

```python
# ❌ Problem: Need to JOIN data from different shards
# Users on Shard 1, Orders on Shard 2 (different user)

def get_all_orders_for_product(product_id):
    """
    Need to query all shards and merge results
    """
    all_orders = []
    
    for shard in shards:
        cur = shard.cursor()
        cur.execute("""
            SELECT * FROM orders 
            WHERE product_id = %s
        """, (product_id,))
        all_orders.extend(cur.fetchall())
    
    return all_orders

# ✅ Solution: Denormalize data or use application-level JOINs
```

**2. Distributed Transactions:**

```python
# ❌ Problem: Need to update data on multiple shards atomically

def transfer_money(from_user_id, to_user_id, amount):
    """
    If users are on different shards, how to ensure atomicity?
    """
    from_shard = get_shard(from_user_id)
    to_shard = get_shard(to_user_id)
    
    if from_shard == to_shard:
        # Easy: Single transaction
        from_shard.execute("BEGIN")
        from_shard.execute("UPDATE accounts SET balance = balance - %s WHERE user_id = %s", (amount, from_user_id))
        from_shard.execute("UPDATE accounts SET balance = balance + %s WHERE user_id = %s", (amount, to_user_id))
        from_shard.execute("COMMIT")
    else:
        # Hard: Two-phase commit (complex!)
        # Or use eventual consistency
        pass

# ✅ Solution: 2PC, Saga pattern, or design to avoid cross-shard transactions
```

**3. Resharding:**

```python
# ❌ Problem: Need to add more shards as data grows

# Original: 4 shards (user_id % 4)
# New: 8 shards (user_id % 8)

# Need to migrate data:
# - User 4 was on shard 0 (4 % 4 = 0)
# - Now should be on shard 4 (4 % 8 = 4)

# ✅ Solution: Consistent hashing or logical shard mapping
```

**4. Shard Key Selection:**

```sql
-- ❌ Bad shard key: Causes hot spots
-- Shard by timestamp -> all new writes go to one shard

-- ✅ Good shard key: Even distribution
-- Shard by user_id hash -> writes distributed evenly

-- ❌ Bad: Can't query without shard key
-- "Get all orders for product X" -> must query all shards

-- ✅ Good: Queries include shard key
-- "Get all orders for user X" -> query one shard
```

---

**Citus: Sharding Extension for PostgreSQL**

```sql
-- Citus adds distributed tables to PostgreSQL
CREATE EXTENSION citus;

-- Add worker nodes (shards)
SELECT citus_add_node('shard1.example.com', 5432);
SELECT citus_add_node('shard2.example.com', 5432);
SELECT citus_add_node('shard3.example.com', 5432);

-- Create distributed table (automatically sharded)
CREATE TABLE users (
    user_id BIGSERIAL PRIMARY KEY,
    name TEXT,
    email TEXT
);

-- Distribute table by user_id
SELECT create_distributed_table('users', 'user_id');

-- Citus automatically:
-- - Shards data across nodes
-- - Routes queries to correct shards
-- - Handles cross-shard queries
-- - Rebalances data when adding nodes

-- Query as normal (Citus handles distribution)
INSERT INTO users (name, email) VALUES ('John', 'john@example.com');
SELECT * FROM users WHERE user_id = 123;

-- Check distribution
SELECT * FROM citus_shards;
```

---

**Combining Strategies:**

**1. Partitioning + Replication:**

```sql
-- Common pattern: Partition large tables, replicate for HA

-- PRIMARY SERVER
CREATE TABLE orders (
    order_id BIGSERIAL,
    order_date DATE NOT NULL,
    user_id INT,
    total_amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- Setup streaming replication to replicas
-- Replicas have same partitioned structure

-- Result:
-- - Fast queries (partition pruning)
-- - High availability (replicas for failover)
-- - Read scaling (distribute reads to replicas)
```

**2. Sharding + Replication:**

```sql
-- Each shard has its own replicas

-- Shard 1 Primary + 2 Replicas
-- Shard 2 Primary + 2 Replicas
-- Shard 3 Primary + 2 Replicas

-- Result:
-- - Write scaling (across shards)
-- - Read scaling (across replicas within each shard)
-- - High availability (replica failover per shard)
```

**3. Partitioning + Sharding:**

```sql
-- Each shard contains partitioned tables

-- Shard 1: Users 1-1M, orders partitioned by date
-- Shard 2: Users 1M-2M, orders partitioned by date
-- Shard 3: Users 2M-3M, orders partitioned by date

-- Result:
-- - Massive scalability
-- - Query performance (partition pruning)
-- - Complexity (very high!)
```

---

**Real-World Decision Making:**

**Scenario 1: E-commerce Startup (1M users, 10M orders)**

```
Choice: Partitioning + Replication

Why:
- Data fits on one server (with partitioning)
- Need HA (replication for failover)
- Read-heavy (replicas for read scaling)
- No need for sharding complexity yet

Setup:
- Partition orders by date (monthly)
- 1 primary + 2 replicas
- Read/write splitting in application
```

**Scenario 2: Social Media Platform (500M users, 100B posts)**

```
Choice: Sharding + Replication

Why:
- Massive dataset (can't fit on one server)
- Both reads and writes need scaling
- User-centric data (shard by user_id)

Setup:
- 100 shards (by user_id hash)
- Each shard has 1 primary + 2 replicas
- Citus or custom sharding logic
- Denormalize data to avoid cross-shard queries
```

**Scenario 3: Analytics Platform (1B events/day)**

```
Choice: Partitioning (TimescaleDB)

Why:
- Time-series data (partition by time)
- Mostly append-only writes (one server handles it)
- Queries are time-based (partition pruning helps)
- Old data can be compressed/archived

Setup:
- Partition by day using TimescaleDB
- Compression for old data
- One powerful server (or add replicas for HA)
```

**Scenario 4: Multi-Tenant SaaS (10K tenants)**

```
Choice: Sharding by tenant_id

Why:
- Data isolation per tenant
- Each tenant's queries stay within one shard
- Can dedicate shards to large tenants

Setup:
- Shard by tenant_id hash
- Large tenants get dedicated shards
- Small tenants share shards
- Each shard has replicas for HA
```

---

**Decision Framework:**

```
Start with: Single PostgreSQL instance
↓
Growing data (>100GB): Add Partitioning
↓
Read-heavy workload: Add Replication (1 primary + N replicas)
↓
Write-heavy workload: Consider Sharding
↓
Massive scale (>1TB, >10K writes/sec): Sharding + Replication + Partitioning
```

**When to use each:**

| Use Case | Solution | Why |
|----------|----------|-----|
| Large table (>100GB) | Partitioning | Faster queries, easier maintenance |
| High availability | Replication | Failover, zero downtime |
| Read-heavy | Replication | Scale reads across replicas |
| Write-heavy | Sharding | Distribute writes across shards |
| Geographic distribution | Replication or Geo-Sharding | Low latency for users |
| Regulatory compliance | Sharding by geography | Data residency requirements |
| Multi-tenant isolation | Sharding by tenant | Data isolation, dedicated resources |

---

**Key Takeaways:**

1. **Partitioning** - Same server, split tables for performance
   - ✅ Easy to implement
   - ✅ Transparent to applications
   - ❌ Doesn't scale writes across servers

2. **Replication** - Multiple servers, same data
   - ✅ High availability (failover)
   - ✅ Read scaling (distribute reads)
   - ❌ Doesn't scale writes (all writes go to primary)

3. **Sharding** - Multiple servers, different data
   - ✅ Scales reads AND writes
   - ✅ True horizontal scaling
   - ❌ Complex (cross-shard queries, distributed transactions)

4. **Combine strategies** for maximum benefit:
   - Partitioning + Replication (most common)
   - Sharding + Replication (large scale)
   - All three (extreme scale)

5. **Start simple, scale gradually**:
   - Begin with single server + partitioning
   - Add replication when needed
   - Shard only when absolutely necessary

6. **Choose shard key carefully**:
   - Even distribution
   - Queries should include shard key
   - Avoid cross-shard transactions

7. **PostgreSQL native support**:
   - Partitioning: Built-in, mature
   - Replication: Built-in, streaming or logical
   - Sharding: Requires extensions (Citus) or custom logic

---

