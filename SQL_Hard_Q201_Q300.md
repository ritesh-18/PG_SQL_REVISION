# PostgreSQL Interview Questions — HARD (Q201–Q300)

> Each question includes: **What it does**, **Key concept**, and the **PostgreSQL query**.

---

### Q201. Recursive CTE — full org hierarchy
**What it does:** Traverses the employee table as a tree, showing every employee's depth and full path from the root manager.
**Key concept:** `WITH RECURSIVE` needs two parts: (1) Anchor — the starting rows (WHERE manager_id IS NULL); (2) Recursive — joins back to the CTE to find children. Depth and path are computed incrementally at each level.
```sql
WITH RECURSIVE org_tree AS (
    SELECT id, name, manager_id, 0 AS level, name::TEXT AS path
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.name, e.manager_id, t.level + 1,
           t.path || ' > ' || e.name
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id
)
SELECT id, name, level, path FROM org_tree ORDER BY path;
```

---

### Q202. Recursive CTE — subtree under a given manager
**What it does:** Fetches a manager (id=5) and all their direct and indirect reports.
**Key concept:** The anchor selects only the root node (id=5). The recursive part finds all employees whose manager_id is in the current result set. The recursion stops when no new employees are found.
```sql
WITH RECURSIVE subtree AS (
    SELECT id, name, manager_id FROM employees WHERE id = 5
    UNION ALL
    SELECT e.id, e.name, e.manager_id
    FROM employees e
    JOIN subtree s ON e.manager_id = s.id
)
SELECT * FROM subtree;
```

---

### Q203. Recursive CTE — Fibonacci sequence
**What it does:** Generates the Fibonacci sequence up to 1000.
**Key concept:** Both `a` and `b` are carried forward — `a` becomes the previous `b`, and `b` becomes `a+b`. This is a pure recursive computation, not a table traversal.
```sql
WITH RECURSIVE fib(a, b) AS (
    SELECT 0, 1
    UNION ALL
    SELECT b, a + b FROM fib WHERE b < 1000
)
SELECT a AS fibonacci FROM fib;
```

---

### Q204. Gaps and Islands — consecutive date ranges
**What it does:** Groups consecutive sales dates into ranges (islands), finding the start and end of each streak.
**Key concept:** Islands technique: `date - ROW_NUMBER()` is constant within a consecutive sequence. GROUP BY that constant groups consecutive dates together. A gap breaks the sequence, producing a new group value.
```sql
WITH sales_days AS (
    SELECT DISTINCT sale_date FROM sales ORDER BY sale_date
),
grouped AS (
    SELECT sale_date,
      sale_date - ROW_NUMBER() OVER (ORDER BY sale_date)::INT AS grp
    FROM sales_days
)
SELECT MIN(sale_date) AS range_start, MAX(sale_date) AS range_end,
       COUNT(*) AS days_in_range
FROM grouped
GROUP BY grp
ORDER BY range_start;
```

---

### Q205. Find sequence gaps
**What it does:** Identifies ranges of missing IDs in the employees table.
**Key concept:** Self-join to the nearest greater ID, then filter where the gap between them is more than 1. For large tables, using `generate_series` + LEFT JOIN is more efficient.
```sql
SELECT s.id + 1 AS gap_start,
       n.id - 1 AS gap_end,
       n.id - s.id - 1 AS gap_size
FROM employees s
JOIN employees n ON n.id = (
    SELECT MIN(id) FROM employees WHERE id > s.id
)
WHERE n.id > s.id + 1;
```

---

### Q206. Sessionize user events
**What it does:** Groups orders into "sessions" where any gap of more than 30 minutes starts a new session.
**Key concept:** Session boundary detection: if the gap from the previous order > 30 minutes (or it's the first order), increment a session counter. SUM of these flags partitioned per customer creates unique session IDs.
```sql
WITH diffs AS (
    SELECT customer_id, order_date,
      order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS diff
    FROM orders
),
sessions AS (
    SELECT customer_id, order_date,
      SUM(CASE WHEN diff > INTERVAL '30 minutes' OR diff IS NULL THEN 1 ELSE 0 END)
        OVER (PARTITION BY customer_id ORDER BY order_date) AS session_id
    FROM diffs
)
SELECT customer_id, session_id, MIN(order_date), MAX(order_date), COUNT(*) AS events
FROM sessions
GROUP BY customer_id, session_id;
```

---

### Q207. GROUPING SETS
**What it does:** Returns department totals, year totals, department+year totals, and a grand total — all in one query.
**Key concept:** `GROUPING SETS ((a), (b), (a,b), ())` — equivalent to running four separate GROUP BY queries and UNION ALLing them. More efficient and expressive than separate queries.
```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS yr, COUNT(*) AS headcount
FROM employees
GROUP BY GROUPING SETS (
    (department),
    (EXTRACT(YEAR FROM hire_date)),
    (department, EXTRACT(YEAR FROM hire_date)),
    ()
);
```

---

### Q208. ROLLUP — subtotals and grand total
**What it does:** Creates subtotals per department, then per year within department, then a grand total.
**Key concept:** `ROLLUP(a, b)` is shorthand for GROUPING SETS: (a,b), (a), () — it aggregates from most granular to least. Used in financial reports, sales summaries.
```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS yr,
       SUM(salary) AS total_salary
FROM employees
GROUP BY ROLLUP(department, EXTRACT(YEAR FROM hire_date))
ORDER BY department, yr;
```

---

### Q209. CUBE — all dimension combinations
**What it does:** Produces every possible combination of grouping for region and year.
**Key concept:** `CUBE(a, b)` = GROUPING SETS of all 2^n subsets: (a,b), (a), (b), (). Used in data warehouses for OLAP-style cube analysis across multiple dimensions.
```sql
SELECT region, EXTRACT(YEAR FROM sale_date) AS yr,
       SUM(amount) AS total
FROM sales
GROUP BY CUBE(region, EXTRACT(YEAR FROM sale_date));
```

---

### Q210. GROUPING() function
**What it does:** Distinguishes subtotal rows from regular rows using a flag (1 = it was grouped away, 0 = real value).
**Key concept:** `GROUPING(col)` returns 1 when that column was aggregated away in a ROLLUP/CUBE/GROUPING SETS row. Use it to label subtotal/grand-total rows differently in reports.
```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS yr,
       SUM(salary),
       GROUPING(department) AS is_dept_total,
       GROUPING(EXTRACT(YEAR FROM hire_date)) AS is_yr_total
FROM employees
GROUP BY ROLLUP(department, EXTRACT(YEAR FROM hire_date));
```

---

### Q211. JSONB — basic insert, read, and navigate
**What it does:** Stores JSON metadata in a product row, then reads specific fields using `->>` and `->`.
**Key concept:** `->` returns JSONB; `->>` returns TEXT. `->` can chain: `col->'specs'->>'ram'`. JSONB is binary-stored, indexed, and supports containment/existence operators. JSON is text-only.
```sql
UPDATE products SET metadata = '{"color":"red","weight":1.5,"tags":["sale","new"]}' WHERE id=1;
SELECT name, metadata->>'color' AS color FROM products;
SELECT name, (metadata->'specs'->>'ram') AS ram FROM products;
```

---

### Q212. JSONB — containment with @>
**What it does:** Finds products whose metadata contains the sub-document `{"tags":["sale"]}`.
**Key concept:** `@>` (contains) operator checks if the left JSONB contains the right JSONB. Works deeply — the right side must be a subset. Indexed efficiently by a GIN index.
```sql
SELECT * FROM products WHERE metadata @> '{"tags": ["sale"]}';
```

---

### Q213. JSONB — key existence operators
**What it does:** Tests if a JSONB object has a specific key or any/all of a set of keys.
**Key concept:** `?` = key exists; `?|` = any key from array exists; `?&` = all keys from array exist. These are more efficient than extracting and comparing the value.
```sql
SELECT * FROM products WHERE metadata ? 'color';
SELECT * FROM products WHERE metadata ?| ARRAY['color','weight'];
SELECT * FROM products WHERE metadata ?& ARRAY['color','weight'];
```

---

### Q214. JSONB — update a nested field
**What it does:** Changes only the `color` field inside the metadata JSON without replacing the whole object.
**Key concept:** `jsonb_set(target, path, new_value, create_if_missing)` — surgically updates a nested key. Path is a text array like `'{color}'` or `'{specs,ram}'`. The fourth argument (true) creates the key if absent.
```sql
UPDATE products
SET metadata = jsonb_set(metadata, '{color}', '"blue"', true)
WHERE id = 1;
```

---

### Q215. JSONB — remove a key
**What it does:** Deletes the `color` key from the metadata object.
**Key concept:** `jsonb - 'key'` removes a top-level key. `jsonb #- '{path,to,key}'` removes a nested key. Non-destructive — does not affect other keys.
```sql
UPDATE products SET metadata = metadata - 'color' WHERE id = 1;
```

---

### Q216. JSONB — aggregate rows into an object
**What it does:** Creates a single JSON object mapping employee names to their salaries.
**Key concept:** `jsonb_object_agg(key, value)` — aggregates key-value pairs from multiple rows into one JSONB object. Useful for building API response payloads directly in SQL.
```sql
SELECT jsonb_object_agg(name, salary) AS emp_salary_map
FROM employees WHERE department = 'Engineering';
```

---

### Q217. JSONB — build array from rows
**What it does:** Creates a JSON array of objects, one per Engineering employee.
**Key concept:** `jsonb_agg(jsonb_build_object(...))` — two functions combined. `jsonb_build_object` creates a JSONB object from alternating key/value args. `jsonb_agg` collects them into an array.
```sql
SELECT jsonb_agg(jsonb_build_object('name', name, 'salary', salary))
FROM employees WHERE department = 'Engineering';
```

---

### Q218. JSONB — expand object to key-value rows
**What it does:** Converts a JSONB object's key-value pairs into separate rows.
**Key concept:** `jsonb_each(json)` is a set-returning function. Each key-value pair becomes a row. Use `jsonb_each_text()` for TEXT values instead of JSONB.
```sql
SELECT key, value FROM products, jsonb_each(metadata) WHERE id = 1;
```

---

### Q219. JSONB — filter inside a JSON array
**What it does:** Finds products that have a tag value of 'new' anywhere in their tags array.
**Key concept:** `jsonb_array_elements_text()` — expands a JSON array into rows. Combined with EXISTS, this checks for membership without needing a GIN index. With a GIN index, use `@>` instead.
```sql
SELECT * FROM products
WHERE EXISTS (
    SELECT 1 FROM jsonb_array_elements_text(metadata->'tags') AS t
    WHERE t = 'new'
);
```

---

### Q220. Full-Text Search — basic usage
**What it does:** Searches product descriptions for documents containing both 'wireless' and 'keyboard'.
**Key concept:** `to_tsvector` converts text to a searchable document. `to_tsquery` creates a query. `@@` is the match operator. `&` = AND, `|` = OR, `!` = NOT, `<->` = phrase (adjacent words).
```sql
SELECT name, description
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'wireless & keyboard');
```

---

### Q221. Full-Text Search — GIN index
**What it does:** Adds a stored tsvector column and GIN index for fast full-text queries.
**Key concept:** Storing the tsvector in a column avoids recomputing it at query time. A GIN (Generalized Inverted Index) index inverts the token list — a lookup finds all rows containing a token in O(log n).
```sql
ALTER TABLE products ADD COLUMN search_vec TSVECTOR;
UPDATE products SET search_vec = to_tsvector('english', COALESCE(name,'') || ' ' || COALESCE(description,''));
CREATE INDEX idx_products_fts ON products USING GIN(search_vec);
SELECT name FROM products WHERE search_vec @@ plainto_tsquery('english', 'wireless keyboard');
```

---

### Q222. Full-Text Search — ranking
**What it does:** Returns search results ordered by relevance score.
**Key concept:** `ts_rank(tsvector, tsquery)` — scoring function based on lexeme frequency and position. Higher rank = better match. `ts_rank_cd` adds cover density (proximity bonus). Wrap in ORDER BY for ranked results.
```sql
SELECT name,
  ts_rank(to_tsvector('english', description), query) AS rank
FROM products,
     to_tsquery('english', 'wireless & keyboard') AS query
WHERE to_tsvector('english', description) @@ query
ORDER BY rank DESC;
```

---

### Q223. PL/pgSQL — function with conditional and row count
**What it does:** Updates salaries for a department by a percentage, validates input, and returns the affected row count.
**Key concept:** `GET DIAGNOSTICS var = ROW_COUNT` captures affected rows. `RAISE EXCEPTION` throws an error with a formatted message. `$$ ... $$` is the function body delimiter (dollar quoting).
```sql
CREATE OR REPLACE FUNCTION give_raise(dept TEXT, pct NUMERIC)
RETURNS INTEGER AS $$
DECLARE
    emp_count INTEGER := 0;
BEGIN
    IF pct <= 0 OR pct > 100 THEN
        RAISE EXCEPTION 'Percentage must be between 0 and 100';
    END IF;
    UPDATE employees SET salary = salary * (1 + pct / 100)
    WHERE department = dept AND is_active = TRUE;
    GET DIAGNOSTICS emp_count = ROW_COUNT;
    RETURN emp_count;
END;
$$ LANGUAGE plpgsql;

SELECT give_raise('Engineering', 10);
```

---

### Q224. PL/pgSQL — FOR loop over query
**What it does:** Iterates over distinct departments and performs per-department logic.
**Key concept:** `FOR rec IN SELECT ... LOOP ... END LOOP` — PL/pgSQL cursor-like iteration. `RECORD` type adapts to whatever the SELECT returns. `RAISE NOTICE` prints to the PostgreSQL log/client.
```sql
CREATE OR REPLACE FUNCTION sync_dept_heads() RETURNS VOID AS $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT DISTINCT department FROM employees LOOP
        RAISE NOTICE 'Processing: %', rec.department;
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

### Q225. TRIGGER — audit salary changes
**What it does:** After any salary UPDATE, inserts a log record into an audit table.
**Key concept:** `AFTER UPDATE ... FOR EACH ROW` — fires once per updated row. `OLD` = row before update; `NEW` = row after. `RETURNS TRIGGER` is mandatory for trigger functions. Return value from AFTER triggers is ignored.
```sql
CREATE TABLE employees_audit (
    operation CHAR(1), changed_at TIMESTAMP DEFAULT NOW(),
    old_salary NUMERIC, new_salary NUMERIC, employee_id INT
);
CREATE OR REPLACE FUNCTION log_salary_change() RETURNS TRIGGER AS $$
BEGIN
    IF OLD.salary <> NEW.salary THEN
        INSERT INTO employees_audit(operation, old_salary, new_salary, employee_id)
        VALUES ('U', OLD.salary, NEW.salary, OLD.id);
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER trg_salary_audit
AFTER UPDATE ON employees
FOR EACH ROW EXECUTE FUNCTION log_salary_change();
```

---

### Q226. TRIGGER — prevent salary decrease
**What it does:** Raises an error if anyone tries to reduce an employee's salary.
**Key concept:** `BEFORE UPDATE ... FOR EACH ROW` — fires before the row is written. BEFORE triggers can modify `NEW` or raise exceptions to cancel the operation. RAISE EXCEPTION rolls back the current statement.
```sql
CREATE OR REPLACE FUNCTION prevent_salary_cut() RETURNS TRIGGER AS $$
BEGIN
    IF NEW.salary < OLD.salary THEN
        RAISE EXCEPTION 'Salary cannot be decreased. Old: %, New: %', OLD.salary, NEW.salary;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER trg_no_salary_cut
BEFORE UPDATE ON employees
FOR EACH ROW EXECUTE FUNCTION prevent_salary_cut();
```

---

### Q227. Row-Level Security (RLS) — employees see only themselves
**What it does:** Restricts employees to only read their own row, while admins bypass the policy.
**Key concept:** RLS adds a WHERE clause to every query automatically based on `USING` expression. `current_setting('app.current_user_id')` is an application-level variable set per connection. FORCE ROW LEVEL SECURITY applies to table owners too.
```sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;
CREATE POLICY emp_self_view ON employees FOR SELECT
USING (id = current_setting('app.current_user_id')::INT);
ALTER TABLE employees FORCE ROW LEVEL SECURITY;
GRANT SELECT ON employees TO employee_role;
```

---

### Q228. Range partitioning by date
**What it does:** Creates a partitioned table where each child covers one calendar year.
**Key concept:** `PARTITION BY RANGE (col)` — PostgreSQL routes INSERTs to the correct partition automatically based on the partitioning column's value. Partition pruning skips irrelevant partitions in queries with date filters.
```sql
CREATE TABLE sales_partitioned (
    id SERIAL, sale_date DATE NOT NULL, amount NUMERIC(12,2), region VARCHAR(100)
) PARTITION BY RANGE (sale_date);
CREATE TABLE sales_2023 PARTITION OF sales_partitioned FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');
CREATE TABLE sales_2024 PARTITION OF sales_partitioned FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

---

### Q229. List partitioning by region
**What it does:** Routes rows to different partitions based on the region column's value.
**Key concept:** `PARTITION BY LIST` — values are explicitly listed per partition. A DEFAULT partition catches all values not listed elsewhere. Use for low-cardinality categorical data.
```sql
CREATE TABLE sales_by_region (
    id SERIAL, region VARCHAR(100) NOT NULL, amount NUMERIC(12,2)
) PARTITION BY LIST (region);
CREATE TABLE sales_north PARTITION OF sales_by_region FOR VALUES IN ('North','Northeast');
CREATE TABLE sales_south PARTITION OF sales_by_region FOR VALUES IN ('South','Southeast');
CREATE TABLE sales_default PARTITION OF sales_by_region DEFAULT;
```

---

### Q230. Hash partitioning
**What it does:** Distributes rows evenly across 4 partitions based on a hash of customer_id.
**Key concept:** `PARTITION BY HASH` — evenly distributes data when no natural range/list structure exists. MODULUS = total partition count; REMAINDER identifies which partition. Good for write-heavy workloads.
```sql
CREATE TABLE orders_hashed (
    id SERIAL, customer_id INT, amount NUMERIC(12,2)
) PARTITION BY HASH (customer_id);
CREATE TABLE orders_hash_0 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_hash_1 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_hash_2 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_hash_3 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

### Q231. LATERAL — top 3 products per category
**What it does:** Returns the 3 most expensive products from each category.
**Key concept:** `CROSS JOIN LATERAL` — for each row from the left (each category), the LATERAL subquery runs with the left row in scope. Equivalent to a correlated subquery but in FROM, allowing LIMIT per group.
```sql
SELECT p.category, t.name, t.price
FROM (SELECT DISTINCT category FROM products) p
CROSS JOIN LATERAL (
    SELECT name, price FROM products
    WHERE category = p.category
    ORDER BY price DESC LIMIT 3
) t;
```

---

### Q232. LATERAL — revenue in the last 90 days per customer
**What it does:** Shows each customer's recent spending without a subquery in SELECT.
**Key concept:** LATERAL allows aggregation in the FROM clause with access to outer row values. More efficient than a SELECT subquery when the result is used in multiple places.
```sql
SELECT c.name AS customer, last_orders.total_recent
FROM customers c
CROSS JOIN LATERAL (
    SELECT SUM(amount) AS total_recent
    FROM orders
    WHERE customer_id = c.id
      AND order_date >= CURRENT_DATE - INTERVAL '90 days'
) last_orders;
```

---

### Q233. Window frame — ROWS vs RANGE
**What it does:** Compares physical-row-based and value-based window frames on the same data.
**Key concept:** `ROWS BETWEEN n PRECEDING AND CURRENT ROW` counts physical rows. `RANGE BETWEEN INTERVAL '3 days' PRECEDING AND CURRENT ROW` includes all rows within 3 days of the current row's date, regardless of row count.
```sql
SELECT sale_date, amount,
  SUM(amount) OVER (ORDER BY sale_date ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS rows_sum,
  SUM(amount) OVER (ORDER BY sale_date RANGE BETWEEN INTERVAL '3 days' PRECEDING AND CURRENT ROW) AS range_sum
FROM sales;
```

---

### Q234. Median per department
**What it does:** Computes the median salary separately for each department.
**Key concept:** `PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary)` is an ordered-set aggregate. It can be used with GROUP BY to produce per-group medians. Cannot be used as a window function.
```sql
SELECT department,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY department;
```

---

### Q235. Salary anomaly detection — Z-score
**What it does:** Finds employees whose salary is more than 2 standard deviations from the mean (statistical outliers).
**Key concept:** Z-score = (value - mean) / stddev. Values with |Z| > 2 or > 3 are considered outliers. `NULLIF(sigma, 0)` prevents division by zero if all salaries are equal.
```sql
WITH stats AS (
    SELECT AVG(salary) AS mu, STDDEV(salary) AS sigma FROM employees
)
SELECT name, salary,
  ROUND(((salary - mu) / NULLIF(sigma, 0))::NUMERIC, 2) AS z_score
FROM employees, stats
WHERE ABS((salary - mu) / NULLIF(sigma, 0)) > 2;
```

---

### Q236. Unpivot using VALUES
**What it does:** Manually creates a tall (normalized) dataset from a wide (denormalized) format.
**Key concept:** `VALUES` as a table source — each row is a literal row. Used to inline small data sets or to unpivot when crosstab is not needed. Also useful in tests.
```sql
SELECT region, quarter, sales
FROM (VALUES
  ('North','Q1',100000), ('North','Q2',150000),
  ('South','Q1',90000),  ('South','Q2',110000)
) AS t(region, quarter, sales);
```

---

### Q237. Pivot using crosstab (tablefunc)
**What it does:** Converts rows into columns — one column per year — using the crosstab function.
**Key concept:** `crosstab()` requires the `tablefunc` extension. The first argument is a query returning (row_id, category, value); the second is the list of category values to become columns. Column types must be explicitly declared.
```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;
SELECT * FROM crosstab(
    'SELECT department, EXTRACT(YEAR FROM hire_date)::TEXT, COUNT(*)
     FROM employees GROUP BY 1, 2 ORDER BY 1, 2',
    'SELECT DISTINCT EXTRACT(YEAR FROM hire_date)::TEXT FROM employees ORDER BY 1'
) AS ct(department TEXT, "2022" BIGINT, "2023" BIGINT, "2024" BIGINT);
```

---

### Q238. Transitive closure — all reachable graph nodes
**What it does:** Given a connections table, finds all nodes reachable from any starting node (full closure of a directed graph).
**Key concept:** Recursive CTE on a graph — the anchor is the direct edges, and the recursive step extends the path by one hop. The `visited` array (or path tracking) prevents infinite cycles.
```sql
WITH RECURSIVE reachable(from_id, to_id) AS (
    SELECT from_id, to_id FROM connections
    UNION
    SELECT r.from_id, c.to_id FROM reachable r JOIN connections c ON r.to_id = c.from_id
)
SELECT DISTINCT from_id, to_id FROM reachable ORDER BY from_id, to_id;
```

---

### Q239. DISTINCT ON — latest record per group
**What it does:** Returns the most recent order per customer in a single pass.
**Key concept:** `DISTINCT ON (col)` — PostgreSQL's most efficient "latest per group" technique. ORDER BY must list the DISTINCT ON column first. Processes rows in order and keeps only the first occurrence.
```sql
SELECT DISTINCT ON (customer_id)
    customer_id, order_date, amount, status
FROM orders
ORDER BY customer_id, order_date DESC;
```

---

### Q240. Detect value changes using IS DISTINCT FROM
**What it does:** Filters rows where the department changed from the previous row.
**Key concept:** `IS DISTINCT FROM` — like `<>` but NULL-safe: `NULL IS DISTINCT FROM NULL` returns FALSE. Equivalent to `(a <> b OR a IS NULL OR b IS NULL)`. Essential when comparing nullable columns.
```sql
SELECT name, department,
  LAG(department) OVER (PARTITION BY id ORDER BY hire_date) AS prev_dept
FROM employees
WHERE department IS DISTINCT FROM LAG(department) OVER (PARTITION BY id ORDER BY hire_date);
```

---

### Q241. Weighted average
**What it does:** Calculates salary average weighted by the headcount in each department.
**Key concept:** A simple AVG of department averages gives equal weight to each department. A weighted average accounts for department sizes. Computed as SUM(dept_avg × headcount) / SUM(headcount).
```sql
WITH dept_stats AS (
    SELECT department, AVG(salary) AS dept_avg, COUNT(*) AS cnt
    FROM employees GROUP BY department
)
SELECT ROUND(SUM(dept_avg * cnt) / SUM(cnt), 2) AS weighted_avg_salary
FROM dept_stats;
```

---

### Q242. Table inheritance
**What it does:** Creates a parent `vehicles` table and child tables `cars` and `trucks` that inherit its columns.
**Key concept:** PostgreSQL table inheritance — child tables include all parent columns plus their own. `SELECT * FROM vehicles` returns rows from all child tables. `SELECT * FROM ONLY cars` excludes inherited tables.
```sql
CREATE TABLE vehicles (id SERIAL PRIMARY KEY, make VARCHAR(100), model VARCHAR(100), year INT);
CREATE TABLE cars (wheel_count INT DEFAULT 4) INHERITS (vehicles);
CREATE TABLE trucks (payload_tons NUMERIC) INHERITS (vehicles);
SELECT * FROM vehicles;
SELECT * FROM ONLY cars;
```

---

### Q243. Advisory locks
**What it does:** Acquires and releases application-level named locks using integer keys.
**Key concept:** Advisory locks are not tied to any table or row — they're named by an integer. Applications use them to coordinate distributed access (e.g., "only one worker processes job type 42 at a time").
```sql
SELECT pg_try_advisory_lock(42);
SELECT pg_advisory_unlock(42);
SELECT pg_advisory_xact_lock(42); -- auto-released on commit/rollback
```

---

### Q244. FOR UPDATE — pessimistic row lock
**What it does:** Reads and locks a specific row so no other transaction can modify it until commit.
**Key concept:** `FOR UPDATE` acquires a row-level exclusive lock. Other transactions trying to SELECT FOR UPDATE the same row will wait. Use for "read-modify-write" sequences where you can't tolerate concurrent modifications.
```sql
BEGIN;
SELECT * FROM employees WHERE id = 1 FOR UPDATE;
UPDATE employees SET salary = 95000 WHERE id = 1;
COMMIT;
```

---

### Q245. SKIP LOCKED — non-blocking queue processing
**What it does:** Multiple workers each grab different pending orders without waiting for each other.
**Key concept:** `FOR UPDATE SKIP LOCKED` — skips rows that are already locked by another transaction instead of waiting. The foundation of work-queue patterns in PostgreSQL (e.g., job queues, task processors).
```sql
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY order_date
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

---

### Q246. SAVEPOINT — nested transaction rollback
**What it does:** Rolls back only the second insert (which failed) while keeping the first.
**Key concept:** `SAVEPOINT name` creates a named point within a transaction. `ROLLBACK TO SAVEPOINT name` undoes work since that point without rolling back the whole transaction. Used for partial-failure handling.
```sql
BEGIN;
INSERT INTO employees (name, salary) VALUES ('Vikram Singh', 60000);
SAVEPOINT after_vikram;
INSERT INTO employees (name, salary) VALUES ('Invalid', -1);
ROLLBACK TO SAVEPOINT after_vikram;
COMMIT;
```

---

### Q247. EXPLAIN with BUFFERS
**What it does:** Shows memory (shared_buffers) and disk I/O usage alongside the query plan.
**Key concept:** `BUFFERS` option shows: shared hit (cache), read (disk), written (dirty), dirtied. High "read" numbers on large tables indicate the data isn't cached — consider adding memory or indexes.
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.name, d.name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.salary > 80000;
```

---

### Q248. Covering index (index-only scan)
**What it does:** Creates an index that includes all columns needed by the query so the heap is never accessed.
**Key concept:** `INCLUDE (col1, col2)` — adds non-key columns to the index leaf pages. PostgreSQL can answer the query entirely from the index without touching the table (Index Only Scan). Much faster for selective queries.
```sql
CREATE INDEX idx_emp_dept_name ON employees(department) INCLUDE (name, salary);
EXPLAIN SELECT name, salary FROM employees WHERE department = 'Engineering';
-- Should show: Index Only Scan
```

---

### Q249. GIN index on JSONB
**What it does:** Creates a GIN index that makes containment queries on JSONB very fast.
**Key concept:** A regular B-Tree index can't index inside JSONB. GIN (Generalized Inverted Index) creates a posting list for every key and value. Makes `@>`, `?`, `?|`, `?&` operators indexed and fast.
```sql
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);
SELECT * FROM products WHERE metadata @> '{"color":"red"}';
```

---

### Q250. GIN index for full-text search
**What it does:** Creates a GIN index on the tsvector expression for text search.
**Key concept:** GIN indexes the individual lexemes of tsvector documents. A query using `@@` can probe the index for matching documents instantly instead of scanning all rows.
```sql
CREATE INDEX idx_products_fts ON products USING GIN(to_tsvector('english', description));
EXPLAIN SELECT * FROM products WHERE to_tsvector('english', description) @@ plainto_tsquery('english','wireless');
```

---

### Q251. BRIN index for sequential data
**What it does:** Creates a Block Range INdex on the sale_date column — ideal for append-only time-series data.
**Key concept:** BRIN stores min/max values per range of disk blocks, not per row. Much smaller than B-Tree. Effective only when column values correlate with physical insertion order (natural ordering). Fast for range scans on large, naturally ordered tables.
```sql
CREATE INDEX idx_sales_date_brin ON sales USING BRIN(sale_date);
```

---

### Q252. pg_stat_statements — slowest queries
**What it does:** Finds the 10 most expensive queries by total execution time.
**Key concept:** `pg_stat_statements` tracks execution statistics for all distinct queries. Requires `shared_preload_libraries = 'pg_stat_statements'` in postgresql.conf. Essential for performance tuning in production.
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
SELECT query, calls, total_exec_time / 1000 AS total_sec,
       mean_exec_time / 1000 AS mean_sec, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 10;
```

---

### Q253. Table bloat detection
**What it does:** Identifies tables with a high ratio of dead tuples — candidates for VACUUM.
**Key concept:** PostgreSQL MVCC leaves dead tuples after UPDATE/DELETE. `n_dead_tup` shows unvacuumed row versions. High dead_pct means wasted space and slower scans. Run VACUUM to reclaim. AUTOVACUUM handles this automatically but may lag under heavy load.
```sql
SELECT relname AS table,
  pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
  n_dead_tup, n_live_tup,
  ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables ORDER BY dead_pct DESC NULLS LAST;
```

---

### Q254. VACUUM and ANALYZE
**What it does:** Reclaims dead tuple space and updates query planner statistics.
**Key concept:** `VACUUM` marks dead tuples as reusable (non-blocking). `VACUUM FULL` rewrites the table (exclusive lock). `ANALYZE` collects statistics used by the query planner for cost estimation. Stale stats cause bad query plans.
```sql
VACUUM employees;
VACUUM FULL employees;
ANALYZE employees;
VACUUM ANALYZE employees;
```

---

### Q255. Find unused indexes
**What it does:** Lists indexes that have never been used by any query.
**Key concept:** `pg_stat_user_indexes.idx_scan` counts how many times an index was used to answer queries. Indexes with `idx_scan = 0` are unused — they slow down writes but help no reads. Safe to drop after verification.
```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY tablename, indexname;
```

---

### Q256. Find tables needing indexes
**What it does:** Identifies large tables being sequentially scanned far more than index-scanned.
**Key concept:** `pg_stat_user_tables.seq_scan` vs `idx_scan` — if seq_scan >> idx_scan on a large table, a missing index is likely causing full table scans. The `seq_tup_read` column shows the total rows read this way.
```sql
SELECT relname, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan AND n_live_tup > 10000
ORDER BY seq_tup_read DESC;
```

---

### Q257. Active connections breakdown
**What it does:** Shows how many connections exist in each state per user and application.
**Key concept:** `pg_stat_activity` — real-time view of all backend processes. States: 'active' (running query), 'idle' (waiting for client), 'idle in transaction' (dangerous — holding locks). Use this to diagnose connection leaks.
```sql
SELECT state, COUNT(*) AS connections, usename, application_name
FROM pg_stat_activity
GROUP BY state, usename, application_name
ORDER BY connections DESC;
```

---

### Q258. Kill long-running queries
**What it does:** Finds queries running for over 5 minutes and terminates them.
**Key concept:** `pg_terminate_backend(pid)` sends SIGTERM (graceful). `pg_cancel_backend(pid)` sends SIGINT (cancel current query but keep connection). Long-running queries hold locks and block other operations.
```sql
SELECT pid, now() - query_start AS duration, query
FROM pg_stat_activity
WHERE query != '<IDLE>' AND query_start < now() - INTERVAL '5 minutes';
SELECT pg_terminate_backend(pid);
SELECT pg_cancel_backend(pid);
```

---

### Q259. Detect blocking queries
**What it does:** Shows which sessions are blocking other sessions and the queries involved.
**Key concept:** `pg_blocking_pids(pid)` — returns an array of PIDs blocking the given process. Joining pg_stat_activity for both blocked and blocker shows what query is causing and experiencing the block.
```sql
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

---

### Q260. Index correlation
**What it does:** Shows how well the physical row order matches the index order for each column.
**Key concept:** `correlation` in `pg_stats` — ranges from -1 to 1. If 1.0, rows are stored in the same order as the index (ideal for range scans). If 0, random order means the index causes random I/O for large range queries (bitmap scan is better).
```sql
SELECT tablename, attname, correlation
FROM pg_stats WHERE tablename = 'employees'
ORDER BY ABS(correlation) DESC;
```

---

### Q261. Dynamic SQL with EXECUTE
**What it does:** Builds a SQL query dynamically using table and column names as parameters.
**Key concept:** `EXECUTE format(...)` — runs a dynamically constructed SQL string. `%I` quotes identifiers (table/column names) safely. `%L` quotes literal values. Prevents SQL injection in dynamic queries.
```sql
CREATE OR REPLACE FUNCTION get_count_by(tbl TEXT, col TEXT, val TEXT)
RETURNS BIGINT AS $$
DECLARE result BIGINT;
BEGIN
    EXECUTE format('SELECT COUNT(*) FROM %I WHERE %I = $1', tbl, col)
    INTO result USING val;
    RETURN result;
END;
$$ LANGUAGE plpgsql;
SELECT get_count_by('employees', 'department', 'Engineering');
```

---

### Q262. Scheduled jobs with pg_cron
**What it does:** Schedules a materialized view refresh every hour and VACUUM every night.
**Key concept:** `pg_cron` — PostgreSQL extension for cron-style job scheduling inside the database. Jobs run in the database's own background worker. Cron syntax: minute hour day month weekday.
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;
SELECT cron.schedule('refresh-stats', '0 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY dept_salary_stats');
SELECT cron.schedule('nightly-vacuum', '0 2 * * *', 'VACUUM ANALYZE employees');
```

---

### Q263. Custom aggregate function
**What it does:** Creates an aggregate that multiplies all values in a group (product instead of sum).
**Key concept:** Custom aggregates need a state transition function (SFUNC) and a state type (STYPE). Each row updates the state. `INITCOND` is the starting state. Used when built-in aggregates don't cover a needed computation.
```sql
CREATE OR REPLACE FUNCTION salary_product_state(state NUMERIC, sal NUMERIC)
RETURNS NUMERIC AS $$ SELECT COALESCE(state, 1) * sal; $$ LANGUAGE SQL;
CREATE AGGREGATE product_agg(NUMERIC) (
    SFUNC = salary_product_state, STYPE = NUMERIC, INITCOND = '1'
);
SELECT department, product_agg(salary) FROM employees GROUP BY department;
```

---

### Q264. Function returning a TABLE
**What it does:** Returns a result set (multiple rows and columns) from a PL/pgSQL function.
**Key concept:** `RETURNS TABLE(col type, ...)` — the function acts as a table-valued function. `RETURN QUERY SELECT ...` fills the result set. Call it with `SELECT * FROM dept_summary('...')`.
```sql
CREATE OR REPLACE FUNCTION dept_summary(dept_name TEXT)
RETURNS TABLE(emp_name TEXT, emp_salary NUMERIC, rank_in_dept BIGINT) AS $$
BEGIN
    RETURN QUERY
    SELECT name, salary, RANK() OVER (ORDER BY salary DESC)
    FROM employees WHERE department = dept_name;
END;
$$ LANGUAGE plpgsql;
SELECT * FROM dept_summary('Engineering');
```

---

### Q265. Event trigger — log DDL changes
**What it does:** Fires a function whenever any DDL statement (CREATE, DROP, ALTER) is executed.
**Key concept:** `EVENT TRIGGER` fires on database-level events, not row/table events. Events: `ddl_command_start`, `ddl_command_end`, `sql_drop`, `table_rewrite`. Useful for schema change auditing and governance.
```sql
CREATE OR REPLACE FUNCTION log_ddl_event() RETURNS EVENT_TRIGGER AS $$
BEGIN
    RAISE NOTICE 'DDL event: % on %', tg_event, tg_tag;
END;
$$ LANGUAGE plpgsql;
CREATE EVENT TRIGGER ddl_logger ON ddl_command_start EXECUTE FUNCTION log_ddl_event();
```

---

### Q266. Foreign Data Wrapper (FDW)
**What it does:** Connects to a remote PostgreSQL database and queries its tables as if they were local.
**Key concept:** `postgres_fdw` — an FDW that maps remote tables into local "foreign tables." After IMPORT FOREIGN SCHEMA, you can JOIN remote and local tables. The planner pushes down conditions to the remote server.
```sql
CREATE EXTENSION postgres_fdw;
CREATE SERVER remote_db FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote-host', port '5432', dbname 'analytics');
CREATE USER MAPPING FOR current_user SERVER remote_db
OPTIONS (user 'remote_user', password 'secret');
IMPORT FOREIGN SCHEMA public FROM SERVER remote_db INTO local_schema;
SELECT * FROM local_schema.remote_employees;
```

---

### Q267. Logical replication
**What it does:** Streams row-level changes from a source database to a subscriber database in real time.
**Key concept:** Logical replication streams changes at the row/DML level (unlike physical replication which is byte-for-byte). Publisher exposes a PUBLICATION; subscriber creates a SUBSCRIPTION. Changes propagate asynchronously.
```sql
-- On source:
CREATE PUBLICATION emp_pub FOR TABLE employees, orders;
-- On destination:
CREATE SUBSCRIPTION emp_sub
CONNECTION 'host=source-db port=5432 dbname=mydb user=replicator password=secret'
PUBLICATION emp_pub;
```

---

### Q268. COPY — bulk load and export
**What it does:** Loads a CSV file into a table at maximum speed, bypassing row-by-row processing.
**Key concept:** `COPY` is the fastest way to load data in PostgreSQL. It uses a binary protocol between the server and the file system. Bypasses triggers. `\copy` (psql client command) works for client-side files.
```sql
COPY employees(name, department, salary, hire_date)
FROM '/tmp/employees.csv'
WITH (FORMAT CSV, HEADER TRUE, DELIMITER ',');
COPY (SELECT * FROM employees WHERE is_active = TRUE)
TO '/tmp/active_employees.csv'
WITH (FORMAT CSV, HEADER TRUE);
```

---

### Q269. Range type — prevent double booking
**What it does:** Stores booking time ranges and uses an EXCLUSION constraint to prevent any two bookings for the same room from overlapping.
**Key concept:** `TSRANGE` — a range of timestamps. `&&` = overlap operator. `EXCLUDE USING GIST (col1 WITH =, col2 WITH &&)` creates a constraint that rejects any INSERT where both conditions are true simultaneously.
```sql
CREATE TABLE room_reservations (
    id SERIAL PRIMARY KEY, room_id INT, period TSRANGE,
    EXCLUDE USING GIST (room_id WITH =, period WITH &&)
);
INSERT INTO room_reservations(room_id, period)
VALUES (101, '[2024-06-01 10:00, 2024-06-01 12:00)');
```

---

### Q270. Range type operations
**What it does:** Demonstrates overlap check, containment, and union of date ranges.
**Key concept:** Range operators: `&&` overlap, `@>` contains, `<@` contained by, `+` union (if adjacent/overlapping), `-` difference, `*` intersection. Ranges can be open `()` or closed `[]` at each end.
```sql
SELECT '[2024-01-01, 2024-06-30]'::DATERANGE && '[2024-04-01, 2024-12-31]'::DATERANGE AS overlaps;
SELECT '[2024-01-01, 2024-12-31]'::DATERANGE @> '2024-06-15'::DATE AS contains;
SELECT '[2024-01-01, 2024-06-30]'::DATERANGE + '[2024-05-01, 2024-12-31]'::DATERANGE AS union_range;
```

---

### Q271. Generated columns
**What it does:** Automatically computes total_price as unit_price × quantity whenever a row is inserted or updated.
**Key concept:** `GENERATED ALWAYS AS (expr) STORED` — a computed column whose value is automatically maintained by PostgreSQL. Cannot be written to directly. `STORED` means the value is physically saved (not computed on read like a view).
```sql
CREATE TABLE order_summary (
    id SERIAL PRIMARY KEY,
    unit_price NUMERIC(10,2),
    quantity INT,
    total_price NUMERIC(12,2) GENERATED ALWAYS AS (unit_price * quantity) STORED
);
INSERT INTO order_summary(unit_price, quantity) VALUES (250.00, 3);
SELECT * FROM order_summary; -- total_price = 750.00
```

---

### Q272. Custom domain types
**What it does:** Creates reusable constrained types for email, phone, and salary — then uses them in a table definition.
**Key concept:** `CREATE DOMAIN` — a named wrapper around a base type with CHECK constraints. All columns using the domain automatically get the same validation. Changing the domain constraint updates all columns using it.
```sql
CREATE DOMAIN email_address AS TEXT
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
CREATE DOMAIN phone_number AS TEXT CHECK (VALUE ~ '^\d{10}$');
CREATE DOMAIN positive_salary AS NUMERIC CHECK (VALUE > 0);
CREATE TABLE staff (
    id SERIAL PRIMARY KEY, email email_address,
    phone phone_number, salary positive_salary
);
```

---

### Q273. ENUM type
**What it does:** Creates a type-safe status column using an enumeration instead of a raw VARCHAR.
**Key concept:** `CREATE TYPE ... AS ENUM (...)` — values are stored as integers internally (compact), but display as strings. Ordering follows the declared order, not alphabetical. Adding values requires `ALTER TYPE ... ADD VALUE`.
```sql
CREATE TYPE order_status AS ENUM ('pending','processing','shipped','delivered','cancelled');
ALTER TABLE orders ALTER COLUMN status TYPE order_status USING status::order_status;
ALTER TYPE order_status ADD VALUE 'returned' AFTER 'delivered';
```

---

### Q274. Composite type
**What it does:** Defines a reusable `address` struct and stores it as a single column.
**Key concept:** `CREATE TYPE name AS (field type, ...)` — a row type. Access fields with `(column).field` notation. Can be nested in arrays: `address[]`. Used to bundle related fields without a separate table.
```sql
CREATE TYPE address AS (street TEXT, city TEXT, state TEXT, pincode TEXT);
CREATE TABLE customers_v2 (id SERIAL PRIMARY KEY, name TEXT, address address);
INSERT INTO customers_v2(name, address) VALUES ('Priya Nair', ROW('123 MG Road','Bangalore','Karnataka','560001'));
SELECT name, (address).city, (address).pincode FROM customers_v2;
```

---

### Q275. Window — salary share within department
**What it does:** Shows each employee's salary, their department's total, and their percentage share.
**Key concept:** `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` — the full partition frame. Without this, LAST_VALUE and full-partition SUM would only see rows up to the current row.
```sql
SELECT name, department, salary,
  SUM(salary) OVER (PARTITION BY department
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS dept_total,
  ROUND(salary * 100.0 /
    SUM(salary) OVER (PARTITION BY department
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING), 2) AS pct_share
FROM employees;
```

---

### Q276. Detect skewed partitions
**What it does:** Shows how many rows each partition contains to detect uneven data distribution.
**Key concept:** `tableoid::REGCLASS` converts the internal OID to the partition table name. Skewed partitions (one very large, others tiny) mean queries touching all data still hit one big partition and lose partition pruning benefits.
```sql
SELECT tableoid::REGCLASS AS partition, COUNT(*) AS row_count
FROM sales_partitioned
GROUP BY tableoid ORDER BY row_count DESC;
```

---

### Q277. Fill missing hours of day
**What it does:** Shows order count for all 24 hours including hours with zero orders.
**Key concept:** generate_series(0,23) creates all hours. LEFT JOIN with actual data fills in zeros where no orders existed. The pattern of generating a "spine" of all values and LEFT JOINing is called a calendar or grid join.
```sql
WITH hours AS (SELECT generate_series(0,23) AS hr),
order_counts AS (
    SELECT EXTRACT(HOUR FROM order_date) AS hr, COUNT(*) AS cnt
    FROM orders GROUP BY hr
)
SELECT h.hr, COALESCE(o.cnt, 0) AS orders
FROM hours h LEFT JOIN order_counts o ON h.hr = o.hr
ORDER BY h.hr;
```

---

### Q278. Multi-column statistics
**What it does:** Creates extended statistics so the planner understands correlation between department and salary.
**Key concept:** By default, PostgreSQL assumes column values are independent. If department and salary are correlated (engineers earn more), the planner underestimates selectivity. Extended stats correct this, producing better query plans.
```sql
CREATE STATISTICS stats_emp_dept_sal ON department, salary FROM employees;
ANALYZE employees;
EXPLAIN SELECT * FROM employees WHERE department = 'Engineering' AND salary > 70000;
```

---

### Q279. pg_notify — real-time notifications
**What it does:** Publishes a JSON payload to listeners when a new order is inserted.
**Key concept:** `LISTEN/NOTIFY` is PostgreSQL's pub-sub mechanism. `pg_notify(channel, payload)` sends a message. All sessions LISTENing on that channel receive it. Used to trigger application-side events without polling.
```sql
LISTEN new_order;
SELECT pg_notify('new_order', '{"order_id":42,"amount":1500}');
CREATE OR REPLACE FUNCTION notify_new_order() RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('new_order', row_to_json(NEW)::TEXT);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
CREATE TRIGGER trg_new_order AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION notify_new_order();
```

---

### Q280. Outbox pattern
**What it does:** Atomically writes a business record and a corresponding event to an outbox table in a single transaction.
**Key concept:** The Outbox Pattern guarantees events are published if and only if the business transaction commits. A separate process (CDC or polling worker) reads from the outbox and publishes to a message broker. Eliminates dual-write inconsistency.
```sql
CREATE TABLE outbox (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    topic TEXT NOT NULL, payload JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(), sent_at TIMESTAMP
);
BEGIN;
INSERT INTO orders(customer_id, product_id, amount) VALUES (1, 5, 500);
INSERT INTO outbox(topic, payload) VALUES ('order.created', '{"customer_id":1,"amount":500}');
COMMIT;
```

---

### Q281. Custom sequence
**What it does:** Creates a sequence starting at 1000 and uses it as a column default.
**Key concept:** `CREATE SEQUENCE` gives fine-grained control: start value, increment, min/max, cache size, cycle. `nextval('seq')` fetches and advances the sequence. `currval('seq')` returns last fetched value in the current session.
```sql
CREATE SEQUENCE invoice_seq START 1000 INCREMENT 1 CACHE 10;
CREATE TABLE invoices (
    id BIGINT DEFAULT nextval('invoice_seq') PRIMARY KEY,
    amount NUMERIC(12,2), created_at TIMESTAMP DEFAULT NOW()
);
SELECT nextval('invoice_seq');
ALTER SEQUENCE invoice_seq RESTART WITH 5000;
```

---

### Q282. Conditional INSERT using CTE
**What it does:** Inserts a customer only if they don't already exist, and returns the id either way.
**Key concept:** INSERT ... WHERE NOT EXISTS pattern with CTE and UNION ALL — returns the newly inserted id or the existing id. This avoids a separate SELECT then INSERT (which has a race condition in concurrent environments).
```sql
WITH existing AS (
    SELECT id FROM customers WHERE email = 'priya@example.com'
),
inserted AS (
    INSERT INTO customers(name, email)
    SELECT 'Priya Sharma', 'priya@example.com'
    WHERE NOT EXISTS (SELECT 1 FROM existing)
    RETURNING id
)
SELECT id FROM inserted UNION ALL SELECT id FROM existing;
```

---

### Q283. Chained CTE with RETURNING
**What it does:** Inserts an order and immediately deducts the stock — in one statement.
**Key concept:** `INSERT ... RETURNING` in a CTE makes the inserted row available to subsequent CTEs and the final SELECT. This keeps the stock deduction in the same statement as the insert, reducing round trips.
```sql
WITH new_order AS (
    INSERT INTO orders(customer_id, product_id, quantity, amount) VALUES (1,5,2,1000)
    RETURNING id, product_id, quantity
)
UPDATE products
SET stock_qty = stock_qty - new_order.quantity
FROM new_order
WHERE products.id = new_order.product_id;
```

---

### Q284. Buffer cache hit ratio
**What it does:** Shows what percentage of reads each table is serving from memory (shared_buffers) vs. disk.
**Key concept:** A cache hit ratio below 95% on hot tables indicates the shared_buffers setting may be too small, or the working set is too large to fit in memory. `pg_statio_user_tables` tracks I/O per table.
```sql
SELECT relname,
  heap_blks_hit,
  ROUND(heap_blks_hit::NUMERIC / NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS cache_hit_pct
FROM pg_statio_user_tables ORDER BY heap_blks_read DESC;
```

---

### Q285. Enable parallel query
**What it does:** Configures parallel query settings and checks if PostgreSQL uses multiple workers.
**Key concept:** PostgreSQL can use parallel workers for sequential scans, aggregations, and joins on large tables. `max_parallel_workers_per_gather` controls how many. Setting cost params to 0 forces parallel plans for testing.
```sql
SET max_parallel_workers_per_gather = 4;
SET parallel_tuple_cost = 0;
SET parallel_setup_cost = 0;
EXPLAIN SELECT COUNT(*) FROM orders;
```

---

### Q286. Tablespace — storage management
**What it does:** Creates a tablespace on a fast SSD and moves a table and index to it.
**Key concept:** Tablespaces map to filesystem directories. Moving hot tables/indexes to fast storage (NVMe) while keeping cold data on slower disks is a common performance optimization.
```sql
CREATE TABLESPACE fast_ssd LOCATION '/mnt/ssd/pgdata';
ALTER TABLE orders SET TABLESPACE fast_ssd;
ALTER INDEX idx_orders_customer SET TABLESPACE fast_ssd;
```

---

### Q287. Customer inactivity gap analysis
**What it does:** Finds customers who went silent for 90+ days between orders.
**Key concept:** LAG + LEAD on order dates per customer — the difference between consecutive orders reveals gaps. Large gaps indicate churn risk. This feeds retention dashboards and re-engagement campaigns.
```sql
WITH customer_activity AS (
    SELECT customer_id, order_date,
      LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order
    FROM orders
)
SELECT customer_id, order_date AS last_before_gap,
  next_order AS returned_on, next_order - order_date AS gap_days
FROM customer_activity
WHERE next_order - order_date > 90
ORDER BY gap_days DESC;
```

---

### Q288. Longest consecutive ordering streak
**What it does:** Finds each customer's maximum run of consecutive days with an order.
**Key concept:** Islands technique → streak lengths per customer → MAX of those lengths. Two-step: first find all islands (consecutive groups), then find the longest island per customer.
```sql
WITH ranked AS (
    SELECT customer_id, order_date,
      order_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)::INT AS grp
    FROM (SELECT DISTINCT customer_id, order_date::DATE FROM orders) t
),
streaks AS (
    SELECT customer_id, grp, COUNT(*) AS streak_len
    FROM ranked GROUP BY customer_id, grp
)
SELECT customer_id, MAX(streak_len) AS longest_streak
FROM streaks GROUP BY customer_id ORDER BY longest_streak DESC;
```

---

### Q289. Batch UPDATE — zero-downtime backfill
**What it does:** Backfills a new column in small batches to avoid locking the entire table.
**Key concept:** Large UPDATEs on millions of rows hold locks and generate huge WAL. Batching with `pg_sleep` between batches keeps the table responsive. Essential for zero-downtime schema migrations in production.
```sql
DO $$
DECLARE last_id INT := 0;
BEGIN
    LOOP
        UPDATE orders SET delivery_at = order_date + INTERVAL '3 days'
        WHERE id > last_id AND delivery_at IS NULL
        RETURNING MAX(id) INTO last_id;
        EXIT WHEN NOT FOUND;
        PERFORM pg_sleep(0.01);
    END LOOP;
END $$;
```

---

### Q290. Zero-downtime NOT NULL column addition
**What it does:** Safely adds a NOT NULL column to a live production table without locking it.
**Key concept:** Three-step pattern: (1) Add nullable — instant. (2) Backfill existing rows. (3) Add NOT VALID CHECK then VALIDATE (validates without full scan in PG 11+, then alter to NOT NULL). Each step is non-blocking.
```sql
ALTER TABLE orders ADD COLUMN delivery_at TIMESTAMP;
-- (Backfill as in Q289)
ALTER TABLE orders ADD CONSTRAINT delivery_at_not_null CHECK (delivery_at IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT delivery_at_not_null;
ALTER TABLE orders ALTER COLUMN delivery_at SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT delivery_at_not_null;
```

---

### Q291. Custom operator
**What it does:** Creates a `>>>` operator that tests if the left salary is more than 1.5x the right value.
**Key concept:** `CREATE OPERATOR` — binds a function to a symbol. Allows domain-specific syntax: `salary >>> 50000` instead of `salary > 50000 * 1.5`. PostgreSQL supports full operator overloading.
```sql
CREATE FUNCTION salary_greater(a NUMERIC, b NUMERIC) RETURNS BOOLEAN AS $$
    SELECT a > b * 1.5;
$$ LANGUAGE SQL;
CREATE OPERATOR >>> (LEFTARG = NUMERIC, RIGHTARG = NUMERIC, FUNCTION = salary_greater);
SELECT name, salary FROM employees WHERE salary >>> 50000;
```

---

### Q292. Advanced ROLLUP with FILTER
**What it does:** Combines ROLLUP subtotals with conditional aggregation in one query for a rich department report.
**Key concept:** FILTER and ROLLUP can be combined freely. The ROLLUP adds subtotal/grand total rows; FILTER restricts which rows each aggregate considers. This avoids separate queries for different slices.
```sql
SELECT department,
  COUNT(*) AS total,
  COUNT(*) FILTER (WHERE salary > 80000) AS high_earners,
  COUNT(*) FILTER (WHERE hire_date > '2023-01-01') AS recent_hires,
  ROUND(AVG(salary), 2) AS avg_salary,
  ROUND(AVG(salary) FILTER (WHERE is_active), 2) AS active_avg
FROM employees
GROUP BY ROLLUP(department);
```

---

### Q293. Shortest path in a graph
**What it does:** Finds the shortest route from Mumbai to Delhi through a connections graph.
**Key concept:** Recursive CTE Dijkstra approximation — each step extends the path by one edge, accumulating total distance. The `visited` array prevents cycles. `ORDER BY distance LIMIT 1` selects the cheapest path found.
```sql
WITH RECURSIVE paths(from_city, to_city, distance, path, visited) AS (
    SELECT from_city, to_city, distance,
           ARRAY[from_city, to_city], ARRAY[from_city, to_city]
    FROM routes WHERE from_city = 'Mumbai'
    UNION ALL
    SELECT p.from_city, r.to_city, p.distance + r.distance,
           p.path || r.to_city, p.visited || r.to_city
    FROM paths p JOIN routes r ON p.to_city = r.from_city
    WHERE NOT r.to_city = ANY(p.visited)
)
SELECT from_city, to_city, distance, path
FROM paths WHERE to_city = 'Delhi'
ORDER BY distance LIMIT 1;
```

---

### Q294. Event sourcing — rebuild state from events
**What it does:** Reconstructs the current state of an entity by aggregating all its historical events.
**Key concept:** Event sourcing stores facts (what happened) not state (current values). To get current state, replay events: sum deposits, apply name changes, etc. The SQL groups all events for one entity and extracts current values.
```sql
SELECT entity_id,
  MAX(CASE WHEN event_type = 'name_changed' THEN payload->>'name' END) AS name,
  SUM(CASE WHEN event_type = 'amount_added' THEN (payload->>'amount')::NUMERIC ELSE 0 END) AS balance,
  MAX(created_at) AS last_event_at
FROM events WHERE entity_id = 42
GROUP BY entity_id;
```

---

### Q295. JSONB — filter and extract from array of objects
**What it does:** Finds products with at least one in-stock variant and lists their available colors.
**Key concept:** `jsonb_array_elements()` expands a JSON array into rows. Combined with aggregation and GROUP BY, you can analyse nested JSON arrays using standard SQL patterns.
```sql
-- Products with any variant in stock:
SELECT p.name FROM products p,
     jsonb_array_elements(p.metadata->'variants') AS v
WHERE (v->>'qty')::INT > 0
GROUP BY p.name;

-- Available colors per product:
SELECT p.name, ARRAY_AGG(v->>'color') AS available_colors
FROM products p,
     jsonb_array_elements(p.metadata->'variants') AS v
WHERE (v->>'qty')::INT > 0
GROUP BY p.name;
```

---

### Q296. Time-series 5-minute buckets
**What it does:** Aggregates sales into 5-minute windows by rounding the timestamp down.
**Key concept:** Modulo arithmetic on EXTRACT(MINUTE) — subtract the remainder after dividing by 5 to snap each timestamp to its 5-minute bucket boundary. TimescaleDB's `time_bucket()` does this more elegantly.
```sql
SELECT DATE_TRUNC('minute', sale_date) -
       INTERVAL '1 minute' * (EXTRACT(MINUTE FROM sale_date)::INT % 5) AS bucket,
       COUNT(*) AS events, SUM(amount) AS total
FROM sales GROUP BY bucket ORDER BY bucket;
```

---

### Q297. Multi-tenant RLS
**What it does:** Ensures each tenant only sees their own rows by applying an automatic row filter.
**Key concept:** RLS with `current_setting('app.tenant_id')` — the application sets this variable on each connection: `SET LOCAL app.tenant_id = '5'`. Every query against `tenant_data` automatically adds `WHERE tenant_id = 5`.
```sql
CREATE TABLE tenant_data (
    id SERIAL PRIMARY KEY, tenant_id INT NOT NULL, payload JSONB
);
ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;
CREATE POLICY tenant_isolation ON tenant_data
USING (tenant_id = current_setting('app.tenant_id')::INT);
```

---

### Q298. CONCURRENT index build
**What it does:** Builds a large index without blocking concurrent INSERT/UPDATE/DELETE operations.
**Key concept:** `CREATE INDEX CONCURRENTLY` performs two table scans and uses a snapshot to avoid blocking writes. Takes longer than a regular build but doesn't acquire an exclusive lock. Cannot be done inside a transaction block.
```sql
SET max_parallel_maintenance_workers = 4;
CREATE INDEX CONCURRENTLY idx_orders_amount_date
ON orders(amount DESC, order_date DESC);
```

---

### Q299. Table bloat recovery without locking
**What it does:** Rewrites a bloated table compactly without ever acquiring an exclusive lock.
**Key concept:** `pg_repack` (extension) rewrites the table in the background using a shadow table + triggers, then does an atomic rename. VACUUM FULL and CLUSTER are simpler alternatives but require an exclusive lock.
```sql
-- pg_repack must be installed as an extension
-- Run from shell: pg_repack -d mydb --table employees

-- Native alternatives (acquire exclusive lock):
CLUSTER employees USING idx_employees_salary;
VACUUM FULL employees;
```

---

### Q300. Cohort retention analysis
**What it does:** Builds a full cohort retention table showing what percentage of each monthly cohort returns in months 0, 1, 2, 3, etc.
**Key concept:** Three steps: (1) Cohort month = first order month per customer. (2) Calculate months_since_join for each subsequent order. (3) Count retained customers per cohort per period and express as % of cohort size. The gold standard of product analytics.
```sql
WITH cohorts AS (
    SELECT customer_id,
           DATE_TRUNC('month', MIN(order_date))::DATE AS cohort_month
    FROM orders GROUP BY customer_id
),
activity AS (
    SELECT o.customer_id, c.cohort_month,
           (DATE_PART('year',o.order_date) - DATE_PART('year',c.cohort_month)) * 12
           + (DATE_PART('month',o.order_date) - DATE_PART('month',c.cohort_month)) AS months_since_join
    FROM orders o JOIN cohorts c ON o.customer_id = c.customer_id
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(DISTINCT customer_id) AS cohort_size
    FROM cohorts GROUP BY cohort_month
),
retention AS (
    SELECT cohort_month, months_since_join,
           COUNT(DISTINCT customer_id) AS retained
    FROM activity GROUP BY cohort_month, months_since_join
)
SELECT r.cohort_month, cs.cohort_size, r.months_since_join, r.retained,
       ROUND(r.retained * 100.0 / cs.cohort_size, 1) AS retention_pct
FROM retention r
JOIN cohort_sizes cs ON r.cohort_month = cs.cohort_month
ORDER BY r.cohort_month, r.months_since_join;
```
