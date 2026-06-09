# PostgreSQL Interview Questions — MEDIUM (Q101–Q200)

> Each question includes: **What it does**, **Key concept**, and the **PostgreSQL query**.

---

### Q101. INNER JOIN employees with departments
**What it does:** Returns only employees that have a matching department record.
**Key concept:** `INNER JOIN` — the most common join. Returns rows where the join condition is TRUE on both sides. Rows with no match on either side are excluded.
```sql
SELECT e.name, e.salary, d.name AS department_name, d.location
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

---

### Q102. LEFT JOIN — keep all employees
**What it does:** Returns all employees; department columns are NULL for those without a department.
**Key concept:** `LEFT JOIN` keeps every row from the left table. If no matching row in the right table, NULLs fill the right columns. Use when the left table is your "driving" table.
```sql
SELECT e.name, d.name AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

---

### Q103. RIGHT JOIN — keep all departments
**What it does:** Returns all departments; employee columns are NULL for empty departments.
**Key concept:** `RIGHT JOIN` is the mirror of LEFT JOIN — all rows from the right table are preserved. Rarely used in practice; usually rewritten as a LEFT JOIN with tables swapped.
```sql
SELECT e.name, d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

---

### Q104. FULL OUTER JOIN
**What it does:** Returns all employees AND all departments, with NULLs on whichever side has no match.
**Key concept:** `FULL OUTER JOIN` = LEFT JOIN + RIGHT JOIN combined. Every unmatched row from both sides appears with NULLs. Useful for reconciliation and finding mismatches.
```sql
SELECT e.name, d.name AS department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

---

### Q105. CROSS JOIN — cartesian product
**What it does:** Produces every possible combination of employee × department.
**Key concept:** `CROSS JOIN` has no ON condition. If employees has 100 rows and departments has 10, result has 1,000 rows. Used intentionally for generating combinations or date grids.
```sql
SELECT e.name AS employee, d.name AS department
FROM employees e
CROSS JOIN departments d;
```

---

### Q106. Self JOIN — employee with manager
**What it does:** Returns each employee alongside their manager's name.
**Key concept:** A self join connects a table to itself using an alias. Here, `e` is the employee and `m` is the same table representing the manager. LEFT JOIN ensures employees without a manager still appear.
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

### Q107. Customer order summary
**What it does:** Shows each customer's order count and total spend.
**Key concept:** LEFT JOIN + GROUP BY — LEFT JOIN ensures customers with zero orders still appear. Aggregates (COUNT, SUM) operate on the joined result.
```sql
SELECT c.name, COUNT(o.id) AS orders, SUM(o.amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```

---

### Q108. Customers with more than 3 orders
**What it does:** Finds loyal customers who have ordered frequently.
**Key concept:** JOIN + GROUP BY + HAVING — use HAVING (not WHERE) to filter on aggregated values. GROUP BY must include all non-aggregated SELECT columns.
```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 3;
```

---

### Q109. Top-selling product
**What it does:** Finds which product has sold the most total units.
**Key concept:** JOIN + SUM + ORDER BY + LIMIT 1 — the pattern for "top N" without window functions. SUM aggregates quantity across all orders for each product.
```sql
SELECT p.name, SUM(o.quantity) AS total_qty
FROM products p
JOIN orders o ON p.id = o.product_id
GROUP BY p.id, p.name
ORDER BY total_qty DESC
LIMIT 1;
```

---

### Q110. UNION — combine two result sets
**What it does:** Merges results from two separate SELECT queries, removing duplicates.
**Key concept:** `UNION` removes duplicate rows. Both SELECTs must have the same number of columns with compatible types. Column names come from the first SELECT.
```sql
SELECT name, salary FROM employees WHERE department = 'Engineering'
UNION
SELECT name, salary FROM employees WHERE department = 'Finance';
```

---

### Q111. UNION ALL — include duplicates
**What it does:** Combines results keeping all rows including duplicates.
**Key concept:** `UNION ALL` is faster than UNION because it skips the deduplication step. Use when you know there are no duplicates or when duplicates are acceptable.
```sql
SELECT department FROM employees WHERE salary > 60000
UNION ALL
SELECT department FROM employees WHERE hire_date > '2023-01-01';
```

---

### Q112. INTERSECT — rows in both sets
**What it does:** Returns only names that appear in BOTH conditions.
**Key concept:** `INTERSECT` keeps rows common to both result sets (like a set intersection). Deduplicates automatically.
```sql
SELECT name FROM employees WHERE salary > 50000
INTERSECT
SELECT name FROM employees WHERE department = 'Engineering';
```

---

### Q113. EXCEPT — rows in first but not second
**What it does:** Returns high earners who are NOT in the HR department.
**Key concept:** `EXCEPT` subtracts the second set from the first (like set difference). Equivalent to: high earners WHERE NOT in HR. Order matters: A EXCEPT B ≠ B EXCEPT A.
```sql
SELECT name FROM employees WHERE salary > 50000
EXCEPT
SELECT name FROM employees WHERE department = 'HR';
```

---

### Q114. Correlated subquery — above department average
**What it does:** Returns employees earning more than the average of their own department.
**Key concept:** Correlated subquery — the inner query references the outer query's row (`e.department`). Executes once per outer row. Powerful but can be slow; often replaced with a CTE or window function.
```sql
SELECT e.name, e.salary, e.department
FROM employees e
WHERE e.salary > (
    SELECT AVG(salary)
    FROM employees
    WHERE department = e.department
);
```

---

### Q115. EXISTS — customers with orders
**What it does:** Returns customers who have placed at least one order.
**Key concept:** `EXISTS (subquery)` returns TRUE as soon as the subquery finds any row. It stops scanning immediately (short-circuit). Faster than `IN` when the subquery is large.
```sql
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

---

### Q116. NOT EXISTS — customers without orders
**What it does:** Returns customers with zero orders — safer alternative to NOT IN.
**Key concept:** `NOT EXISTS` avoids the NULL trap of `NOT IN`. If `orders.customer_id` ever has NULLs, `NOT IN` returns zero rows, but `NOT EXISTS` still works correctly.
```sql
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

---

### Q117. IN with subquery — employees in high-budget departments
**What it does:** Returns employees from departments with budget over 1 million.
**Key concept:** `IN (subquery)` — the subquery returns a set of values; the outer query filters rows matching any value in that set.
```sql
SELECT * FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE budget > 1000000
);
```

---

### Q118. Basic CTE
**What it does:** Creates a named temporary result set and queries it.
**Key concept:** `WITH name AS (SELECT ...) SELECT ...` — CTE (Common Table Expression). Improves readability. Executes once and is available to the main query like a temporary view.
```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
)
SELECT * FROM high_earners ORDER BY salary DESC;
```

---

### Q119. CTE — compare against department average
**What it does:** Uses a CTE to pre-compute department averages, then joins to find above-average earners.
**Key concept:** Multiple CTEs can be chained. This pattern avoids a correlated subquery and can be more readable and sometimes more performant.
```sql
WITH dept_avg AS (
    SELECT department, AVG(salary) AS avg_sal
    FROM employees
    GROUP BY department
)
SELECT e.name, e.salary, d.avg_sal
FROM employees e
JOIN dept_avg d ON e.department = d.department
WHERE e.salary > d.avg_sal;
```

---

### Q120. ROW_NUMBER — sequential row numbers
**What it does:** Assigns a unique sequential integer to each row within each department, ordered by salary.
**Key concept:** `ROW_NUMBER()` always gives unique numbers — no ties. `PARTITION BY` restarts numbering for each partition. `ORDER BY` inside OVER determines the ranking order.
```sql
SELECT name, department, salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
FROM employees;
```

---

### Q121. RANK — handles ties with gaps
**What it does:** Ranks employees by salary, giving the same rank to ties, but skipping the next rank.
**Key concept:** `RANK()` — if two employees both have rank 2, the next rank is 4 (gap). Contrast with DENSE_RANK which gives rank 3 in the same scenario.
```sql
SELECT name, department, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
```

---

### Q122. DENSE_RANK — no gaps in ranking
**What it does:** Ranks employees by salary with no rank gaps after ties.
**Key concept:** `DENSE_RANK()` — ties get the same rank but the next rank is consecutive (no skip). Used when you need "top 3 salary levels" rather than "top 3 individual employees."
```sql
SELECT name, department, salary,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

---

### Q123. Top 1 employee per department
**What it does:** Returns the highest-paid employee from each department.
**Key concept:** ROW_NUMBER() in a CTE/subquery, then filter WHERE rn = 1. This is the standard "get one row per group" pattern. Using RANK() instead would return multiple rows if there's a tie at rank 1.
```sql
WITH ranked AS (
    SELECT *,
      ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;
```

---

### Q124. LAG — compare with previous row
**What it does:** Shows each employee's salary next to the previous employee's salary, and the difference.
**Key concept:** `LAG(column, offset, default)` accesses a preceding row's value. Default offset is 1. Returns NULL for the first row (no preceding row) unless a default is specified.
```sql
SELECT name, salary,
  LAG(salary) OVER (ORDER BY salary) AS prev_salary,
  salary - LAG(salary) OVER (ORDER BY salary) AS diff
FROM employees;
```

---

### Q125. LEAD — compare with next row
**What it does:** Shows what the next employee's salary is.
**Key concept:** `LEAD(column, offset, default)` accesses a following row's value. Returns NULL for the last row. Used for detecting trends and computing deltas to the next period.
```sql
SELECT name, salary,
  LEAD(salary) OVER (ORDER BY salary) AS next_salary
FROM employees;
```

---

### Q126. Running total
**What it does:** Adds a cumulative salary sum ordered by hire date.
**Key concept:** `SUM() OVER (ORDER BY ...)` — without a frame clause, PostgreSQL uses `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` by default, giving a running total.
```sql
SELECT name, hire_date, salary,
  SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees;
```

---

### Q127. Running average
**What it does:** Computes the average salary of all employees up to and including the current row (in hire order).
**Key concept:** Specifying the frame explicitly with `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` makes ROWS behavior clear. ROWS counts physical rows; RANGE uses logical values.
```sql
SELECT name, hire_date, salary,
  ROUND(AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 2) AS running_avg
FROM employees;
```

---

### Q128. FIRST_VALUE — highest salary alongside every row
**What it does:** Shows the department's max salary on every employee row.
**Key concept:** `FIRST_VALUE(expr) OVER (PARTITION BY ... ORDER BY ...)` — returns the value from the first row of each window partition. Without ORDER BY it returns the first arbitrary row.
```sql
SELECT name, department, salary,
  FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_in_dept
FROM employees;
```

---

### Q129. LAST_VALUE — lowest salary in department
**What it does:** Shows the department's minimum salary on every row.
**Key concept:** `LAST_VALUE` requires `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` because the default frame stops at the current row, making LAST_VALUE behave like CURRENT value.
```sql
SELECT name, department, salary,
  LAST_VALUE(salary) OVER (
    PARTITION BY department ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS lowest_in_dept
FROM employees;
```

---

### Q130. NTILE — salary quartiles
**What it does:** Divides employees into 4 equal-sized buckets based on salary rank.
**Key concept:** `NTILE(n)` distributes rows into n groups as evenly as possible. Bucket 1 = lowest, Bucket 4 = highest. Used for percentile-based segmentation.
```sql
SELECT name, salary,
  NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
```

---

### Q131. PERCENT_RANK — relative rank
**What it does:** Shows what fraction of employees have a lower salary.
**Key concept:** `PERCENT_RANK()` = (rank - 1) / (total_rows - 1). Returns 0 for the lowest, 1 for the highest. Result is between 0 and 1.
```sql
SELECT name, salary,
  ROUND(PERCENT_RANK() OVER (ORDER BY salary)::NUMERIC, 4) AS pct_rank
FROM employees;
```

---

### Q132. CUME_DIST — cumulative distribution
**What it does:** Shows what fraction of employees have a salary ≤ the current employee.
**Key concept:** `CUME_DIST()` = rows_with_value_≤_current / total_rows. Always between 0 and 1. Differs from PERCENT_RANK in how it handles ties.
```sql
SELECT name, salary,
  ROUND(CUME_DIST() OVER (ORDER BY salary)::NUMERIC, 4) AS cume_dist
FROM employees;
```

---

### Q133. Multiple CTEs chained
**What it does:** Computes department stats, filters high-budget departments, then joins them.
**Key concept:** Multiple CTEs are separated by commas after WITH. Each CTE can reference earlier ones. Evaluated logically (and often physically) before the main query.
```sql
WITH dept_stats AS (
    SELECT department, AVG(salary) AS avg_sal, MAX(salary) AS max_sal
    FROM employees GROUP BY department
),
high_budget_depts AS (
    SELECT id, name FROM departments WHERE budget > 500000
)
SELECT d.name AS dept, s.avg_sal, s.max_sal
FROM dept_stats s
JOIN high_budget_depts d ON s.department = d.name;
```

---

### Q134. CASE with aggregation — pivot salary bands
**What it does:** Counts employees in each salary band in a single row (pivot).
**Key concept:** Using CASE inside SUM — the CASE returns 1 when the condition matches and 0 otherwise. SUM then counts the trues. This is the traditional pivot technique in SQL.
```sql
SELECT
  SUM(CASE WHEN salary < 40000 THEN 1 ELSE 0 END) AS low,
  SUM(CASE WHEN salary BETWEEN 40000 AND 80000 THEN 1 ELSE 0 END) AS medium,
  SUM(CASE WHEN salary > 80000 THEN 1 ELSE 0 END) AS high
FROM employees;
```

---

### Q135. FILTER clause with aggregation
**What it does:** Computes multiple selective aggregates in one pass.
**Key concept:** `aggregate FILTER (WHERE condition)` — cleaner than nested CASE. Only rows matching the filter are included in that aggregate. PostgreSQL 9.4+. More readable than CASE SUM approach.
```sql
SELECT
  COUNT(*) FILTER (WHERE department = 'Engineering') AS eng_count,
  COUNT(*) FILTER (WHERE salary > 70000) AS high_earners,
  AVG(salary) FILTER (WHERE is_active = TRUE) AS active_avg_sal
FROM employees;
```

---

### Q136. Find duplicate emails
**What it does:** Identifies email addresses that appear more than once.
**Key concept:** GROUP BY + HAVING COUNT > 1 — the standard duplicate detection pattern. HAVING filters after grouping, keeping only groups with multiple rows.
```sql
SELECT email, COUNT(*) AS cnt
FROM employees
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;
```

---

### Q137. Delete duplicates, keep first
**What it does:** Removes duplicate rows, keeping only the minimum (first) id per email.
**Key concept:** DELETE with NOT IN subquery that picks MIN(id). The MIN acts as a "keep this one" selector. For large tables, use a CTE with RETURNING for better performance.
```sql
DELETE FROM employees
WHERE id NOT IN (
    SELECT MIN(id)
    FROM employees
    GROUP BY email
);
```

---

### Q138. UPSERT — insert or update
**What it does:** Inserts a row; if id already exists, updates salary and name instead.
**Key concept:** `ON CONFLICT (column) DO UPDATE SET ...` — PostgreSQL's upsert. `EXCLUDED` refers to the row that was rejected. Atomic: no race condition between SELECT and INSERT.
```sql
INSERT INTO employees (id, name, salary, department)
VALUES (1, 'Rahul Verma', 85000, 'Engineering')
ON CONFLICT (id)
DO UPDATE SET salary = EXCLUDED.salary, name = EXCLUDED.name;
```

---

### Q139. ON CONFLICT DO NOTHING
**What it does:** Silently ignores the insert if a conflict on id occurs.
**Key concept:** `ON CONFLICT DO NOTHING` — no error, no update, just skip. Useful for idempotent inserts where you don't care about the existing row.
```sql
INSERT INTO employees (id, name, salary)
VALUES (1, 'Anita Rao', 70000)
ON CONFLICT (id) DO NOTHING;
```

---

### Q140. INSERT from SELECT
**What it does:** Copies and transforms rows from the existing table into new rows.
**Key concept:** `INSERT INTO ... SELECT ...` — no VALUES clause needed. The SELECT defines the source data. Column count and types must match.
```sql
INSERT INTO employees (name, department, salary)
SELECT name, 'General', salary * 0.9
FROM employees
WHERE is_active = FALSE;
```

---

### Q141. UPDATE using JOIN (UPDATE FROM)
**What it does:** Gives all Engineering employees a 10% raise, joining via departments table.
**Key concept:** `UPDATE ... FROM ... WHERE` — PostgreSQL's multi-table UPDATE. The FROM clause acts like a JOIN. Much faster than a correlated subquery update.
```sql
UPDATE employees e
SET salary = e.salary * 1.10
FROM departments d
WHERE e.department_id = d.id
  AND d.name = 'Engineering';
```

---

### Q142. STRING_AGG — list employees per department
**What it does:** Concatenates all employee names in each department into a comma-separated list.
**Key concept:** `STRING_AGG(value, delimiter ORDER BY ...)` — aggregate that produces a string. Unlike ARRAY_AGG, it returns a TEXT result. Useful for reports and denormalization.
```sql
SELECT department, STRING_AGG(name, ', ' ORDER BY name) AS emp_list
FROM employees
GROUP BY department;
```

---

### Q143. ARRAY_AGG — collect values into an array
**What it does:** Returns an array of employee names per department.
**Key concept:** `ARRAY_AGG(value ORDER BY ...)` — creates a PostgreSQL array. Result type is TEXT[]. Can be filtered with FILTER clause and unnested later with UNNEST().
```sql
SELECT department, ARRAY_AGG(name ORDER BY salary DESC) AS employees
FROM employees
GROUP BY department;
```

---

### Q144. UNNEST an array
**What it does:** Expands an array into individual rows.
**Key concept:** `UNNEST(array)` — the inverse of ARRAY_AGG. Turns one row with an array into multiple rows. Used to normalize denormalized data.
```sql
SELECT UNNEST(ARRAY['Rahul', 'Priya', 'Amit']) AS name;
```

---

### Q145. Generate a series of dates
**What it does:** Produces one row per month between two dates.
**Key concept:** `generate_series(start, end, step)` — PostgreSQL set-returning function. Step can be an interval. Essential for creating date grids, filling gaps, or building calendar tables.
```sql
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 month') AS month;
```

---

### Q146. Generate a series of numbers
**What it does:** Returns integers from 1 to 10, one per row.
**Key concept:** `generate_series(start, end)` with integers — step defaults to 1. Can also go backwards: `generate_series(10, 1, -1)`.
```sql
SELECT generate_series(1, 10) AS num;
```

---

### Q147. Nth highest salary
**What it does:** Returns the 3rd highest salary using OFFSET.
**Key concept:** `DISTINCT ... ORDER BY ... LIMIT 1 OFFSET N-1` — offset skips N-1 rows; LIMIT 1 returns the next one. DISTINCT ensures each unique salary counts once.
```sql
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;
```

---

### Q148. Find gaps in IDs
**What it does:** Identifies missing employee IDs in the sequence.
**Key concept:** Self-referencing subquery with NOT EXISTS. For each ID, check if ID+1 exists. If not, ID+1 is a gap. More efficient approaches use generate_series and LEFT JOIN for large tables.
```sql
SELECT id + 1 AS missing_from
FROM employees e1
WHERE NOT EXISTS (
    SELECT 1 FROM employees e2 WHERE e2.id = e1.id + 1
)
AND id < (SELECT MAX(id) FROM employees);
```

---

### Q149. LATERAL JOIN — last 2 orders per customer
**What it does:** For each customer, fetches their 2 most recent orders.
**Key concept:** `LATERAL` — allows the subquery on the right side to reference columns from the left side. Acts like a correlated subquery but in the FROM clause. More efficient than a correlated subquery in SELECT.
```sql
SELECT c.name, o.order_date, o.amount
FROM customers c
CROSS JOIN LATERAL (
    SELECT * FROM orders
    WHERE customer_id = c.id
    ORDER BY order_date DESC
    LIMIT 2
) o;
```

---

### Q150. Year-over-year growth
**What it does:** Calculates percentage sales growth compared to the previous year.
**Key concept:** LAG() in a CTE: first aggregate by year, then use LAG to access the previous year's total in the same query pass. Formula: (current - prev) / prev * 100.
```sql
WITH yearly AS (
    SELECT EXTRACT(YEAR FROM sale_date) AS yr, SUM(amount) AS total
    FROM sales
    GROUP BY yr
)
SELECT yr, total,
  LAG(total) OVER (ORDER BY yr) AS prev_year,
  ROUND((total - LAG(total) OVER (ORDER BY yr)) / LAG(total) OVER (ORDER BY yr) * 100, 2) AS growth_pct
FROM yearly;
```

---

### Q151. 3-row moving average
**What it does:** Smooths sales data using a rolling window of 3 records.
**Key concept:** `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` — the frame covers the current row and 2 rows before it, giving a 3-point moving average. Widely used in financial and time-series analysis.
```sql
SELECT sale_date, amount,
  ROUND(AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3
FROM sales;
```

---

### Q152. Quarterly sales pivot
**What it does:** Shows each year as one row with Q1–Q4 sales in separate columns.
**Key concept:** FILTER-based pivot — the cleanest PostgreSQL approach for pivoting. No extension needed. Each FILTER restricts which rows the SUM includes.
```sql
SELECT
  EXTRACT(YEAR FROM sale_date) AS year,
  SUM(amount) FILTER (WHERE EXTRACT(QUARTER FROM sale_date) = 1) AS Q1,
  SUM(amount) FILTER (WHERE EXTRACT(QUARTER FROM sale_date) = 2) AS Q2,
  SUM(amount) FILTER (WHERE EXTRACT(QUARTER FROM sale_date) = 3) AS Q3,
  SUM(amount) FILTER (WHERE EXTRACT(QUARTER FROM sale_date) = 4) AS Q4
FROM sales
GROUP BY year;
```

---

### Q153. Employees earning more than their manager
**What it does:** Finds employees with higher salaries than the person managing them.
**Key concept:** Self JOIN on manager_id — a classic hierarchy query. JOIN employees to itself: one alias for the employee, another for the manager. Then compare salaries.
```sql
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager, m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

### Q154. COALESCE with multiple fallbacks
**What it does:** Returns email if available, else phone, else a default string.
**Key concept:** COALESCE takes any number of arguments and returns the first non-NULL. Useful for implementing fallback chains in reporting.
```sql
SELECT name,
  COALESCE(email, phone, 'No contact') AS contact_info
FROM employees;
```

---

### Q155. Orders in last 30 days
**What it does:** Filters orders placed within the past month.
**Key concept:** `CURRENT_DATE - INTERVAL '30 days'` — dynamic date calculation. Avoids hardcoding a date, so the query is correct at any point in time.
```sql
SELECT * FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days';
```

---

### Q156. 3-table JOIN
**What it does:** Shows order details with human-readable customer and product names.
**Key concept:** Chaining multiple JOINs — each adds another table. Aliases (o, c, p) keep the query readable. Order of JOINs doesn't affect correctness; the planner optimizes it.
```sql
SELECT o.id, c.name AS customer, p.name AS product,
       o.quantity, o.amount, o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id;
```

---

### Q157. Months with no sales
**What it does:** Finds calendar months in 2024 that had zero sales.
**Key concept:** generate_series + LEFT JOIN + WHERE NULL — generates all 12 months, left-joins with actual sales months, and filters for NULLs (months that had no match = no sales).
```sql
WITH all_months AS (
    SELECT generate_series('2024-01-01'::DATE, '2024-12-01'::DATE, '1 month')::DATE AS month
),
sales_months AS (
    SELECT DATE_TRUNC('month', sale_date)::DATE AS month FROM sales
)
SELECT a.month
FROM all_months a
LEFT JOIN sales_months s ON a.month = s.month
WHERE s.month IS NULL;
```

---

### Q158. Relational division — customers who ordered ALL products
**What it does:** Finds customers who have ordered every product in the catalog.
**Key concept:** Relational division via COUNT DISTINCT: if a customer's distinct product count equals the total product count, they've ordered everything. A pure-SQL implementation of the "for all" quantifier.
```sql
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = (SELECT COUNT(*) FROM products);
```

---

### Q159. Percentage of total sales per region
**What it does:** Shows each region's share of total revenue.
**Key concept:** `SUM(SUM(amount)) OVER ()` — nested window function: inner SUM aggregates per group, outer SUM OVER () computes the grand total across all groups.
```sql
SELECT region,
  SUM(amount) AS region_sales,
  ROUND(SUM(amount) * 100.0 / SUM(SUM(amount)) OVER (), 2) AS pct_of_total
FROM sales
GROUP BY region;
```

---

### Q160. Named WINDOW clause
**What it does:** Defines a reusable window specification and applies it to multiple functions.
**Key concept:** `WINDOW w AS (...)` at the end of the query — avoids repeating the same OVER clause. All window functions referencing `OVER w` share the same partition/order definition.
```sql
SELECT name, salary,
  RANK()        OVER w AS rnk,
  DENSE_RANK()  OVER w AS dense_rnk,
  ROW_NUMBER()  OVER w AS rn
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

---

### Q161. DISTINCT ON — latest record per group
**What it does:** Returns the most recent sale for each salesperson in one query.
**Key concept:** `DISTINCT ON (col)` — PostgreSQL-specific. Keeps only the first row per distinct value of the specified column. ORDER BY must start with the DISTINCT ON column.
```sql
SELECT DISTINCT ON (salesperson_id)
  salesperson_id, sale_date, amount
FROM sales
ORDER BY salesperson_id, sale_date DESC;
```

---

### Q162. Multi-column UPDATE
**What it does:** Simultaneously updates salary and modifies department name for senior employees.
**Key concept:** `SET col1 = ..., col2 = ...` — multiple assignments in one UPDATE. All changes apply to the same row atomically.
```sql
UPDATE employees
SET salary = salary * 1.05,
    department = 'Senior ' || department
WHERE hire_date < '2020-01-01';
```

---

### Q163. Recursive CTE — number series
**What it does:** Generates numbers from 1 to 10 using recursion.
**Key concept:** `WITH RECURSIVE` — two parts: anchor (base case: n=1) and recursive (n+1 while n<10). PostgreSQL evaluates the anchor first, then repeatedly applies the recursive part until no new rows are produced.
```sql
WITH RECURSIVE nums AS (
    SELECT 1 AS n
    UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10
)
SELECT * FROM nums;
```

---

### Q164. EXTRACT vs DATE_PART
**What it does:** Shows that both functions return the same result.
**Key concept:** `EXTRACT` is SQL standard; `DATE_PART` is PostgreSQL-specific. Both return a FLOAT8. For integer results cast to INTEGER. EXTRACT is preferred for portability.
```sql
SELECT EXTRACT(YEAR FROM NOW()),
       DATE_PART('year', NOW());
```

---

### Q165. TO_TIMESTAMP — string to timestamp
**What it does:** Parses a formatted string into a proper TIMESTAMP.
**Key concept:** `TO_TIMESTAMP(string, format)` — format uses: YYYY (4-digit year), MM (month), DD (day), HH24 (24h hour), MI (minute), SS (second). Opposite of TO_CHAR.
```sql
SELECT TO_TIMESTAMP('2024-06-15 10:30:00', 'YYYY-MM-DD HH24:MI:SS');
```

---

### Q166. Group by week
**What it does:** Aggregates sales totals per calendar week.
**Key concept:** `DATE_TRUNC('week', date)` returns the Monday of each week (ISO standard). Grouping by this truncated value groups all days of the same week together.
```sql
SELECT DATE_TRUNC('week', sale_date) AS week_start,
       SUM(amount) AS weekly_sales
FROM sales
GROUP BY week_start
ORDER BY week_start;
```

---

### Q167. BETWEEN with timestamps
**What it does:** Finds orders placed in the first half of 2024.
**Key concept:** Date BETWEEN for half-year ranges. Note: for TIMESTAMP columns, '2024-06-30' is interpreted as midnight, missing records on the evening of June 30. Use `< '2024-07-01'` for safety.
```sql
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-06-30';
```

---

### Q168. Computed columns — annual salary and tax
**What it does:** Derives annual salary and estimated tax in SELECT without storing them.
**Key concept:** Arithmetic expressions in SELECT create on-the-fly computed columns. They exist only in the result set, not in the table.
```sql
SELECT name, salary,
  ROUND(salary * 12, 2) AS annual_salary,
  ROUND(salary * 0.20, 2) AS tax_estimate
FROM employees;
```

---

### Q169. Window function with GROUP BY result
**What it does:** Shows each employee's salary alongside their department's average and their deviation from it.
**Key concept:** Window functions run AFTER GROUP BY but before the final result. Here used without GROUP BY — the window computes partition aggregates while all rows are still visible.
```sql
SELECT name, department, salary,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg,
  salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

---

### Q170. Employees with matching salary
**What it does:** Finds pairs of employees earning the same amount.
**Key concept:** Self JOIN with `e1.id < e2.id` — prevents duplicates (without it, each pair appears twice and each employee is matched with themselves).
```sql
SELECT e1.name, e2.name, e1.salary
FROM employees e1
JOIN employees e2 ON e1.salary = e2.salary AND e1.id < e2.id;
```

---

### Q171. Orders by day of week
**What it does:** Shows order volume for each day of the week.
**Key concept:** `TO_CHAR(date, 'Day')` returns the full day name. `EXTRACT(DOW ...)` returns 0=Sunday through 6=Saturday. ORDER BY the numeric form for correct weekday ordering.
```sql
SELECT TO_CHAR(order_date, 'Day') AS day_name,
       COUNT(*) AS order_count
FROM orders
GROUP BY TO_CHAR(order_date, 'Day'), EXTRACT(DOW FROM order_date)
ORDER BY EXTRACT(DOW FROM order_date);
```

---

### Q172. Materialized view
**What it does:** Pre-computes and stores department statistics as a physical table snapshot.
**Key concept:** A materialized view stores results physically (unlike a regular view). Fast to query, but data is stale until refreshed. Use `CONCURRENTLY` to refresh without blocking reads.
```sql
CREATE MATERIALIZED VIEW dept_salary_stats AS
SELECT department,
       COUNT(*) AS headcount,
       AVG(salary) AS avg_salary,
       MAX(salary) AS max_salary
FROM employees
GROUP BY department;
```

---

### Q173. Refresh materialized view
**What it does:** Updates the materialized view with current data from the underlying table.
**Key concept:** `REFRESH MATERIALIZED VIEW` — replaces all data in the view. `CONCURRENTLY` avoids an exclusive lock but requires a UNIQUE index on the view.
```sql
REFRESH MATERIALIZED VIEW dept_salary_stats;
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_salary_stats;
```

---

### Q174. Simple SQL function
**What it does:** Creates a reusable function that returns the employee count for a given department.
**Key concept:** `CREATE OR REPLACE FUNCTION ... LANGUAGE SQL` — the simplest function type. SQL functions can only run SQL statements. For control flow (IF, loops), use PL/pgSQL.
```sql
CREATE OR REPLACE FUNCTION get_employee_count(dept TEXT)
RETURNS INTEGER AS $$
    SELECT COUNT(*) FROM employees WHERE department = dept;
$$ LANGUAGE SQL;

SELECT get_employee_count('Engineering');
```

---

### Q175. ILIKE — case-insensitive search
**What it does:** Finds customers named "sharma" in any case variation.
**Key concept:** `ILIKE` is PostgreSQL-specific. In standard SQL, use `LOWER(name) LIKE LOWER(pattern)`. ILIKE doesn't use regular B-Tree indexes; use a trigram index (pg_trgm) for fast ILIKE queries.
```sql
SELECT * FROM customers WHERE name ILIKE '%sharma%';
```

---

### Q176. Regular expression matching
**What it does:** Filters employees whose email is from gmail.com using a regex pattern.
**Key concept:** `~` is case-sensitive regex match; `~*` is case-insensitive. `!~` means does NOT match. PostgreSQL uses POSIX regex syntax.
```sql
SELECT * FROM employees WHERE email ~ '@gmail\.com$';
SELECT * FROM employees WHERE email ~* '@gmail\.com$';
```

---

### Q177. SIMILAR TO
**What it does:** Matches names starting with Rahul, Priya, or Amit.
**Key concept:** `SIMILAR TO` is SQL standard regex. Syntax is between LIKE and full regex: `%` and `_` wildcards still work, but `|` for alternation and `()` for grouping are also supported.
```sql
SELECT * FROM employees WHERE name SIMILAR TO '(Rahul|Priya|Amit)%';
```

---

### Q178. Array containment and overlap
**What it does:** Checks if an array contains a specific element or overlaps with another array.
**Key concept:** `&&` = overlap (any element in common); `@>` = contains (all right elements in left). These operators work with PostgreSQL native arrays and have GIN index support.
```sql
SELECT * FROM products WHERE ARRAY['electronics', 'mobile'] && ARRAY['mobile'];
SELECT ARRAY_LENGTH(ARRAY[1,2,3,4], 1);
```

---

### Q179. String ↔ Array conversion
**What it does:** Splits a delimited string into an array and joins an array back into a string.
**Key concept:** `STRING_TO_ARRAY(text, delimiter)` and `ARRAY_TO_STRING(array, delimiter)` — inverse operations. Useful for parsing CSV-like text columns.
```sql
SELECT STRING_TO_ARRAY('Rahul,Priya,Amit', ',') AS names;
SELECT ARRAY_TO_STRING(ARRAY['Rahul','Priya','Amit'], ' | ') AS name_list;
```

---

### Q180. Window frame with ROWS BETWEEN
**What it does:** Computes a sum of the current and previous 2 rows for each date.
**Key concept:** `ROWS BETWEEN 2 PRECEDING AND CURRENT ROW` — explicit physical row frame. Contrast with RANGE which uses value-based boundaries. ROWS is more predictable when there are duplicate values.
```sql
SELECT sale_date, amount,
  SUM(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  ) AS sliding_3day_sum
FROM sales;
```

---

### Q181. Median salary
**What it does:** Computes the median (middle value) of all salaries.
**Key concept:** `PERCENTILE_CONT(0.5)` returns the 50th percentile with interpolation. `PERCENTILE_DISC(0.5)` returns an actual value from the dataset. Both use `WITHIN GROUP (ORDER BY ...)`.
```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;
```

---

### Q182. 90th percentile salary
**What it does:** Returns the salary below which 90% of employees fall.
**Key concept:** `PERCENTILE_CONT(p)` — p is the fraction (0 to 1). Used in SLA analysis (p99 latency), compensation benchmarking, and performance profiling.
```sql
SELECT PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY salary) AS p90_salary
FROM employees;
```

---

### Q183. Most common salary value
**What it does:** Returns the salary that appears most frequently.
**Key concept:** `MODE() WITHIN GROUP (ORDER BY ...)` — PostgreSQL ordered-set aggregate. Returns the most common value. Useful for finding the "typical" value as an alternative to mean.
```sql
SELECT MODE() WITHIN GROUP (ORDER BY salary) AS most_common_salary
FROM employees;
```

---

### Q184. Standard deviation and variance
**What it does:** Measures salary spread — how far values deviate from the mean.
**Key concept:** `STDDEV()` = population standard deviation; `STDDEV_SAMP()` = sample stddev. A large stddev means wide salary distribution. Use for outlier detection and fairness analysis.
```sql
SELECT
  ROUND(STDDEV(salary)::NUMERIC, 2) AS std_dev,
  ROUND(VARIANCE(salary)::NUMERIC, 2) AS variance
FROM employees;
```

---

### Q185. Orders above 75th percentile
**What it does:** Returns high-value orders (top 25% by amount).
**Key concept:** Scalar subquery returning PERCENTILE_CONT value, used as a threshold in WHERE. This is the "top N percent of values" filtering pattern.
```sql
SELECT * FROM orders
WHERE amount > (
    SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amount)
    FROM orders
);
```

---

### Q186. Consecutive date streaks (islands)
**What it does:** Finds groups of consecutive days on which each customer ordered.
**Key concept:** Island detection — `date - ROW_NUMBER()` is constant within a consecutive sequence. Group by this constant to identify islands. A classic advanced SQL interview question.
```sql
WITH numbered AS (
    SELECT customer_id, order_date,
      order_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)::INT AS grp
    FROM (SELECT DISTINCT customer_id, order_date FROM orders) t
),
streaks AS (
    SELECT customer_id, grp, COUNT(*) AS streak_len,
           MIN(order_date) AS streak_start, MAX(order_date) AS streak_end
    FROM numbered
    GROUP BY customer_id, grp
)
SELECT * FROM streaks WHERE streak_len > 1;
```

---

### Q187. Month-over-month change
**What it does:** Shows how much order volume grew or shrank month to month.
**Key concept:** LAG(1) on monthly aggregates — first aggregate to month level, then apply LAG to compare adjacent months. The difference is the absolute change.
```sql
WITH monthly AS (
    SELECT DATE_TRUNC('month', order_date) AS month, SUM(amount) AS total
    FROM orders GROUP BY month
)
SELECT month, total,
  total - LAG(total) OVER (ORDER BY month) AS mom_change
FROM monthly;
```

---

### Q188. EXPLAIN — show query plan
**What it does:** Displays how PostgreSQL plans to execute the query (without actually running it).
**Key concept:** `EXPLAIN` shows: node type (Seq Scan / Index Scan), cost estimates, row estimates. Used to understand whether indexes are being used. Does NOT execute the query.
```sql
EXPLAIN SELECT * FROM employees WHERE salary > 80000;
```

---

### Q189. EXPLAIN ANALYZE — run and profile
**What it does:** Executes the query and shows actual timing and row counts alongside plan estimates.
**Key concept:** `EXPLAIN ANALYZE` actually runs the query. Key metrics: "actual time=X..Y", "actual rows=N", "loops=N". Discrepancy between "rows=estimated" and "actual rows" reveals stale statistics.
```sql
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Engineering';
```

---

### Q190. Composite index
**What it does:** Creates a multi-column index on department and salary.
**Key concept:** A composite index helps queries filtering on both columns or the leading column only (department). It does NOT help if only salary is filtered. Column order matters.
```sql
CREATE INDEX idx_emp_dept_sal ON employees(department, salary DESC);
```

---

### Q191. Partial index
**What it does:** Creates an index that only covers active employees.
**Key concept:** A partial index has a WHERE clause — it's smaller and faster than a full index because it only indexes qualifying rows. Ideal for high-selectivity subsets (e.g., `is_active = TRUE` is 90% of queries but only 10% of data).
```sql
CREATE INDEX idx_active_emp ON employees(department) WHERE is_active = TRUE;
```

---

### Q192. Tables without primary keys
**What it does:** Lists tables in the public schema that lack a PRIMARY KEY constraint.
**Key concept:** Querying `information_schema.table_constraints` — the SQL standard catalog for constraint metadata. Tables without PKs are harder to replicate and update efficiently.
```sql
SELECT t.table_name
FROM information_schema.tables t
WHERE t.table_schema = 'public'
  AND t.table_type = 'BASE TABLE'
  AND t.table_name NOT IN (
    SELECT ku.table_name
    FROM information_schema.table_constraints tc
    JOIN information_schema.key_column_usage ku ON tc.constraint_name = ku.constraint_name
    WHERE tc.constraint_type = 'PRIMARY KEY'
  );
```

---

### Q193. List indexes on a table
**What it does:** Shows all index names and definitions for the employees table.
**Key concept:** `pg_indexes` — PostgreSQL catalog view. Shows index name, table, and the CREATE INDEX statement. Useful for auditing over-indexing or finding redundant indexes.
```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'employees';
```

---

### Q194. Table size on disk
**What it does:** Returns the total disk space used by the employees table including indexes.
**Key concept:** `pg_total_relation_size(oid)` includes table + TOAST + indexes. `pg_relation_size()` is table heap only. `pg_size_pretty()` formats bytes into human-readable units.
```sql
SELECT pg_size_pretty(pg_total_relation_size('employees')) AS table_size;
```

---

### Q195. List all schemas
**What it does:** Shows all schemas in the current database.
**Key concept:** `information_schema.schemata` — lists all schemas. In multi-tenant apps, each tenant may have their own schema. `public` is the default.
```sql
SELECT schema_name FROM information_schema.schemata;
```

---

### Q196. Create a schema
**What it does:** Creates a new namespace for grouping related tables.
**Key concept:** Schemas are namespaces. Tables with the same name can exist in different schemas. Use `schema.table` notation to specify: `reports.employees`.
```sql
CREATE SCHEMA reports;
```

---

### Q197. Set search path
**What it does:** Tells PostgreSQL to look in `reports` schema first, then `public`.
**Key concept:** `search_path` determines which schema is used when a table name is given without a prefix. Like PATH in a shell. Can be set per session, per role, or per database.
```sql
SET search_path TO reports, public;
```

---

### Q198. Rename a table
**What it does:** Changes the table name from employees to staff.
**Key concept:** `ALTER TABLE ... RENAME TO` — updates the table name catalog entry. All existing indexes, constraints, and views that reference it are automatically updated in PostgreSQL.
```sql
ALTER TABLE employees RENAME TO staff;
```

---

### Q199. Add a CHECK constraint
**What it does:** Enforces that salary is always non-negative.
**Key concept:** `CHECK (expression)` — validates rows on INSERT and UPDATE. If any existing row violates the constraint, the ALTER TABLE fails. Use `NOT VALID` to add without scanning existing rows.
```sql
ALTER TABLE employees
ADD CONSTRAINT chk_salary_positive CHECK (salary >= 0);
```

---

### Q200. Table with multiple constraints
**What it does:** Creates a projects table with SERIAL PK, NOT NULL, CHECK, DEFAULT, and a table-level CONSTRAINT.
**Key concept:** Combining all constraint types — column-level constraints apply to one column; `CONSTRAINT name CHECK (...)` at table level can reference multiple columns (e.g., end_date > start_date).
```sql
CREATE TABLE projects (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(150) NOT NULL,
    budget      NUMERIC(15,2) CHECK (budget > 0),
    start_date  DATE NOT NULL,
    end_date    DATE,
    status      VARCHAR(50) DEFAULT 'planning'
                CHECK (status IN ('planning', 'active', 'completed', 'cancelled')),
    CONSTRAINT chk_dates CHECK (end_date IS NULL OR end_date > start_date)
);
```
