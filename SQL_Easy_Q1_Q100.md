# PostgreSQL Interview Questions — EASY (Q1–Q100)

> Each question includes: **What it does**, **Key concept**, and the **PostgreSQL query**.

---

### Q1. Select all columns from employees
**What it does:** Retrieves every row and every column from the table.
**Key concept:** `SELECT *` — full table scan. In production, always prefer specific column names.
```sql
SELECT * FROM employees;
```

---

### Q2. Select only name and salary
**What it does:** Returns only two columns, reducing data transfer.
**Key concept:** Column projection — fetch only what you need. Reduces network and memory overhead.
```sql
SELECT name, salary FROM employees;
```

---

### Q3. Filter rows by salary
**What it does:** Returns only rows where the salary exceeds 50000.
**Key concept:** `WHERE` clause — filters rows before they are returned. Uses comparison operators: `=`, `>`, `<`, `>=`, `<=`, `<>`.
```sql
SELECT * FROM employees WHERE salary > 50000;
```

---

### Q4. Filter by department
**What it does:** Returns employees belonging to a specific department.
**Key concept:** String equality in WHERE. Note: SQL string comparisons are case-sensitive unless you use `ILIKE` or `LOWER()`.
```sql
SELECT * FROM employees WHERE department = 'Engineering';
```

---

### Q5. Top 5 highest-paid employees
**What it does:** Sorts employees by salary in descending order and returns only the first 5.
**Key concept:** `ORDER BY ... DESC` + `LIMIT`. PostgreSQL processes ORDER BY before LIMIT.
```sql
SELECT * FROM employees ORDER BY salary DESC LIMIT 5;
```

---

### Q6. Count total employees
**What it does:** Returns a single number — the total row count.
**Key concept:** `COUNT(*)` counts all rows including NULLs. `COUNT(column)` excludes NULLs. `AS` gives the result a readable alias.
```sql
SELECT COUNT(*) AS total_employees FROM employees;
```

---

### Q7. Maximum salary
**What it does:** Finds the single highest salary in the table.
**Key concept:** `MAX()` is an aggregate function — collapses all rows into one result. Ignores NULL values.
```sql
SELECT MAX(salary) AS max_salary FROM employees;
```

---

### Q8. Minimum salary
**What it does:** Finds the lowest salary.
**Key concept:** `MIN()` aggregate. Paired with `MAX()` to understand the range of a column.
```sql
SELECT MIN(salary) AS min_salary FROM employees;
```

---

### Q9. Average salary
**What it does:** Computes the mean salary across all employees.
**Key concept:** `AVG()` returns a NUMERIC result. Wrapped in `ROUND(value, 2)` to limit decimal places.
```sql
SELECT ROUND(AVG(salary), 2) AS avg_salary FROM employees;
```

---

### Q10. Sum of all salaries
**What it does:** Adds up all salary values — useful for payroll totals.
**Key concept:** `SUM()` aggregate. NULLs are ignored. Result is NULL if all values are NULL.
```sql
SELECT SUM(salary) AS total_salary FROM employees;
```

---

### Q11. Count employees per department
**What it does:** Groups rows by department and counts employees in each group.
**Key concept:** `GROUP BY` — splits rows into groups. Aggregate functions (COUNT, SUM, AVG) operate per group.
```sql
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department;
```

---

### Q12. Departments with more than 5 employees
**What it does:** Filters groups (not individual rows) by size.
**Key concept:** `HAVING` — like WHERE but applied after GROUP BY. WHERE filters rows; HAVING filters groups.
```sql
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;
```

---

### Q13. Distinct departments
**What it does:** Returns each department name exactly once, removing duplicates.
**Key concept:** `DISTINCT` — deduplicates result rows. Works on one or multiple columns.
```sql
SELECT DISTINCT department FROM employees;
```

---

### Q14. Filter by hire date
**What it does:** Returns employees hired after January 1st, 2020.
**Key concept:** Date comparison in WHERE. Use ISO format `'YYYY-MM-DD'` for date literals in PostgreSQL.
```sql
SELECT * FROM employees WHERE hire_date > '2020-01-01';
```

---

### Q15. Name starts with 'R'
**What it does:** Pattern match — finds names beginning with the letter R.
**Key concept:** `LIKE` — `%` is wildcard for any sequence of characters. `_` matches exactly one character. Case-sensitive in PostgreSQL.
```sql
SELECT * FROM employees WHERE name LIKE 'R%';
```

---

### Q16. Name ends with 'a'
**What it does:** Pattern match for names ending in 'a'.
**Key concept:** `LIKE` with leading `%` — the pattern `%a` means "any characters followed by a".
```sql
SELECT * FROM employees WHERE name LIKE '%a';
```

---

### Q17. Salary between 40000 and 80000
**What it does:** Returns employees whose salary is within the inclusive range.
**Key concept:** `BETWEEN x AND y` is inclusive on both ends. Equivalent to `salary >= 40000 AND salary <= 80000`.
```sql
SELECT * FROM employees WHERE salary BETWEEN 40000 AND 80000;
```

---

### Q18. Employees in HR or Finance
**What it does:** Returns employees belonging to any of the listed departments.
**Key concept:** `IN (list)` — cleaner than multiple `OR` conditions. Equivalent to `department = 'HR' OR department = 'Finance'`.
```sql
SELECT * FROM employees WHERE department IN ('HR', 'Finance');
```

---

### Q19. Employees with no manager
**What it does:** Returns employees where manager_id has no value.
**Key concept:** NULL comparison — you cannot use `= NULL`. Must use `IS NULL`. NULL means "unknown/absent value."
```sql
SELECT * FROM employees WHERE manager_id IS NULL;
```

---

### Q20. Employees with an email address
**What it does:** Returns only employees who have an email on record.
**Key concept:** `IS NOT NULL` — the opposite of IS NULL. Checks for the presence of a value.
```sql
SELECT * FROM employees WHERE email IS NOT NULL;
```

---

### Q21. Sort by hire date ascending
**What it does:** Orders results from the earliest hire to the most recent.
**Key concept:** `ORDER BY column ASC` — ASC is the default direction. ASC = oldest first for dates.
```sql
SELECT * FROM employees ORDER BY hire_date ASC;
```

---

### Q22. Sort by salary then name
**What it does:** Primary sort by salary descending; ties broken alphabetically by name.
**Key concept:** Multi-column ORDER BY — second column only matters when the first has equal values.
```sql
SELECT * FROM employees ORDER BY salary DESC, name ASC;
```

---

### Q23. Insert a new row
**What it does:** Adds a new employee record to the table.
**Key concept:** `INSERT INTO ... VALUES` — you specify the columns and matching values. SERIAL/sequence columns (id) can be omitted if auto-generating.
```sql
INSERT INTO employees (name, department, salary, hire_date, email)
VALUES ('Priya Sharma', 'Engineering', 75000, '2024-01-15', 'priya@example.com');
```

---

### Q24. Update a specific row
**What it does:** Changes the salary of the employee with id = 5.
**Key concept:** `UPDATE ... SET ... WHERE` — without WHERE, it updates every row. Always include a WHERE clause unless intentional.
```sql
UPDATE employees SET salary = 90000 WHERE id = 5;
```

---

### Q25. Delete a specific row
**What it does:** Removes the employee with id = 10.
**Key concept:** `DELETE FROM ... WHERE` — like UPDATE, forgetting WHERE deletes all rows. Use TRUNCATE to clear all rows efficiently.
```sql
DELETE FROM employees WHERE id = 10;
```

---

### Q26. Column alias
**What it does:** Renames a column in the output without changing the table.
**Key concept:** `AS alias_name` — alias is only visible in ORDER BY and the client result. Not usable in WHERE of the same query.
```sql
SELECT name, salary AS monthly_salary FROM employees;
```

---

### Q27. Not in a department
**What it does:** Excludes Engineering employees from results.
**Key concept:** `!=` and `<>` are both "not equal to" in SQL. Note: NULL values are excluded by both — use `OR department IS NULL` if needed.
```sql
SELECT * FROM employees WHERE department != 'Engineering';
```

---

### Q28. Rows with NULL department
**What it does:** Finds employees not assigned to any department.
**Key concept:** NULL filtering. Important for data quality checks and outer join debugging.
```sql
SELECT * FROM employees WHERE department IS NULL;
```

---

### Q29. Uppercase name
**What it does:** Returns names in ALL CAPS.
**Key concept:** `UPPER(text)` — string function. Doesn't modify data in the table. Applied during SELECT only.
```sql
SELECT UPPER(name) AS upper_name FROM employees;
```

---

### Q30. Lowercase name
**What it does:** Returns names in all lowercase.
**Key concept:** `LOWER(text)` — commonly used with `ILIKE` for case-insensitive comparisons: `WHERE LOWER(name) = 'rahul'`.
```sql
SELECT LOWER(name) AS lower_name FROM employees;
```

---

### Q31. Length of name
**What it does:** Returns number of characters in each employee's name.
**Key concept:** `LENGTH(text)` — counts characters, not bytes. For byte length use `OCTET_LENGTH()`.
```sql
SELECT name, LENGTH(name) AS name_length FROM employees;
```

---

### Q32. Concatenate strings
**What it does:** Combines name and department into a single string.
**Key concept:** `||` is the string concatenation operator in PostgreSQL. If any operand is NULL, result is NULL — use `CONCAT()` to handle NULLs safely.
```sql
SELECT name || ' - ' || department AS employee_info FROM employees;
```

---

### Q33. Trim whitespace
**What it does:** Removes leading and trailing spaces from the name.
**Key concept:** `TRIM(text)` removes both ends. `LTRIM()` = left only, `RTRIM()` = right only. Useful for cleaning imported data.
```sql
SELECT TRIM(name) AS trimmed_name FROM employees;
```

---

### Q34. Current date
**What it does:** Returns today's date.
**Key concept:** `CURRENT_DATE` is a PostgreSQL built-in that returns a DATE type (no time component). Useful in WHERE clauses and calculations.
```sql
SELECT CURRENT_DATE;
```

---

### Q35. Current timestamp
**What it does:** Returns the current date and time including timezone.
**Key concept:** `NOW()` returns `TIMESTAMP WITH TIME ZONE`. Use `CURRENT_TIMESTAMP` for the SQL-standard equivalent. Both are evaluated once per transaction.
```sql
SELECT NOW();
```

---

### Q36. Extract year from hire_date
**What it does:** Pulls just the year part from a date column.
**Key concept:** `EXTRACT(field FROM date)` — fields: YEAR, MONTH, DAY, HOUR, MINUTE, DOW (day of week 0=Sunday), DOY (day of year).
```sql
SELECT name, EXTRACT(YEAR FROM hire_date) AS hire_year FROM employees;
```

---

### Q37. Extract month from hire_date
**What it does:** Returns the numeric month (1–12).
**Key concept:** Extracting parts is useful for grouping by month/year without needing to truncate the date.
```sql
SELECT name, EXTRACT(MONTH FROM hire_date) AS hire_month FROM employees;
```

---

### Q38. Employee tenure
**What it does:** Calculates how long each employee has worked.
**Key concept:** `AGE(end, start)` returns an INTERVAL (e.g., "3 years 4 months 12 days"). Called with one arg it uses `NOW()` as end.
```sql
SELECT name, AGE(NOW(), hire_date) AS tenure FROM employees;
```

---

### Q39. Hired in year 2022
**What it does:** Filters employees hired specifically in 2022.
**Key concept:** Combining EXTRACT with WHERE for date-based filtering without needing BETWEEN on full dates.
```sql
SELECT * FROM employees WHERE EXTRACT(YEAR FROM hire_date) = 2022;
```

---

### Q40. First 3 characters of name
**What it does:** Returns a prefix of the name string.
**Key concept:** `LEFT(string, n)` — returns first N characters. Equivalent to `SUBSTRING(name, 1, n)`.
```sql
SELECT name, LEFT(name, 3) AS short_name FROM employees;
```

---

### Q41. Last 3 characters of name
**What it does:** Returns the last 3 characters (suffix).
**Key concept:** `RIGHT(string, n)` — counts from the end. Negative n in SUBSTRING can also achieve this.
```sql
SELECT name, RIGHT(name, 3) AS suffix FROM employees;
```

---

### Q42. Replace substring
**What it does:** Substitutes one string with another within the column value.
**Key concept:** `REPLACE(string, from, to)` — replaces ALL occurrences. Case-sensitive. For regex-based replacement use `REGEXP_REPLACE()`.
```sql
SELECT name, REPLACE(department, 'Engineer', 'Dev') AS dept FROM employees;
```

---

### Q43. Round salary
**What it does:** Rounds salary to 0 decimal places.
**Key concept:** `ROUND(value, precision)` — precision=0 rounds to integer. Negative precision rounds to tens/hundreds: `ROUND(salary, -3)` rounds to nearest thousand.
```sql
SELECT name, ROUND(salary, 0) AS rounded_salary FROM employees;
```

---

### Q44. Absolute value
**What it does:** Returns the non-negative version of a number.
**Key concept:** `ABS(n)` — commonly used with differences to measure distance regardless of direction.
```sql
SELECT ABS(-5000);
```

---

### Q45. Ceiling and floor
**What it does:** Rounds up (CEIL) and rounds down (FLOOR) to nearest integer.
**Key concept:** `CEIL(n)` always rounds up; `FLOOR(n)` always rounds down — even for negatives (FLOOR(-2.3) = -3, not -2).
```sql
SELECT name, CEIL(salary) AS ceil_sal, FLOOR(salary) AS floor_sal FROM employees;
```

---

### Q46. Pagination with LIMIT/OFFSET
**What it does:** Retrieves page 2 assuming 10 records per page.
**Key concept:** `LIMIT n OFFSET m` — skip first M rows, return next N. Used for pagination. Performance degrades at large offsets — use keyset pagination for large tables.
```sql
SELECT * FROM employees ORDER BY id LIMIT 10 OFFSET 10;
```

---

### Q47. Count distinct departments
**What it does:** Counts how many unique departments exist.
**Key concept:** `COUNT(DISTINCT column)` — counts only unique non-NULL values. Different from `COUNT(*)` which counts all rows.
```sql
SELECT COUNT(DISTINCT department) AS dept_count FROM employees;
```

---

### Q48. Average salary per department
**What it does:** Shows mean salary for each department group.
**Key concept:** Combining GROUP BY with AVG. ROUND wraps the aggregate to control decimal output.
```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department;
```

---

### Q49. Departments with high average salary
**What it does:** Only returns departments where mean salary exceeds 60000.
**Key concept:** HAVING filters aggregated groups. Cannot use WHERE here because AVG is computed after rows are grouped.
```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 60000;
```

---

### Q50. NOT BETWEEN
**What it does:** Excludes employees in a salary range.
**Key concept:** `NOT BETWEEN x AND y` — equivalent to `salary < 30000 OR salary > 50000`. Inclusive exclusion on both boundaries.
```sql
SELECT * FROM employees WHERE salary NOT BETWEEN 30000 AND 50000;
```

---

### Q51. Total orders per customer
**What it does:** Counts how many orders each customer has placed.
**Key concept:** Grouping by a foreign key (customer_id) to aggregate related rows.
```sql
SELECT customer_id, COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id;
```

---

### Q52. Total order amount per customer
**What it does:** Sums up the spend per customer.
**Key concept:** SUM + GROUP BY pattern — the foundation of customer lifetime value (CLV) calculations.
```sql
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

---

### Q53. Filter customers by country
**What it does:** Returns customers from India only.
**Key concept:** Basic string WHERE filter. For case-insensitive matching use `ILIKE 'india'` or `LOWER(country) = 'india'`.
```sql
SELECT * FROM customers WHERE country = 'India';
```

---

### Q54. Customers with no orders
**What it does:** Finds customers who have never ordered anything.
**Key concept:** `NOT IN (subquery)` — subquery returns a list of customer_ids that have orders. Caution: if the subquery returns ANY NULL, NOT IN returns nothing — safer to use NOT EXISTS.
```sql
SELECT * FROM customers
WHERE id NOT IN (SELECT DISTINCT customer_id FROM orders);
```

---

### Q55. Cheap products
**What it does:** Returns products priced below 500.
**Key concept:** Numeric comparison in WHERE. For NUMERIC/DECIMAL columns use direct comparison without quoting.
```sql
SELECT * FROM products WHERE price < 500;
```

---

### Q56. Out-of-stock products
**What it does:** Finds products with zero inventory.
**Key concept:** Equality check on numeric column. Useful for inventory management dashboards.
```sql
SELECT * FROM products WHERE stock_qty = 0;
```

---

### Q57. Max price per category
**What it does:** Finds the highest-priced product in each category.
**Key concept:** GROUP BY + MAX() — each category group collapses to the row with the max price. To get the full product row, use a window function or subquery.
```sql
SELECT category, MAX(price) AS max_price
FROM products
GROUP BY category;
```

---

### Q58. Product count per category
**What it does:** Shows how many products exist in each category.
**Key concept:** GROUP BY + COUNT() — a frequency distribution query.
```sql
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category;
```

---

### Q59. Add a column
**What it does:** Adds a new phone column to the employees table.
**Key concept:** `ALTER TABLE ... ADD COLUMN` — in PostgreSQL, adding a nullable column is instant (no table rewrite). Adding NOT NULL without a default requires a table scan in older versions.
```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(15);
```

---

### Q60. Drop a column
**What it does:** Removes the phone column from the table.
**Key concept:** `ALTER TABLE ... DROP COLUMN` — destructive and immediate. The column data is gone. Add `CASCADE` to also drop dependent objects (views, indexes).
```sql
ALTER TABLE employees DROP COLUMN phone;
```

---

### Q61. Rename a column
**What it does:** Renames is_active to active_status.
**Key concept:** `ALTER TABLE ... RENAME COLUMN old TO new` — updates the column name in the schema. Does NOT require a table rewrite.
```sql
ALTER TABLE employees RENAME COLUMN is_active TO active_status;
```

---

### Q62. Set NOT NULL on existing column
**What it does:** Enforces that email must always have a value going forward.
**Key concept:** `SET NOT NULL` — this scans the entire table to verify no current NULLs exist. Will fail if any NULL is found. Do a backfill first.
```sql
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;
```

---

### Q63. Create an index on salary
**What it does:** Creates a B-Tree index to speed up salary-based queries.
**Key concept:** Index = sorted copy of a column stored separately. Speeds up WHERE, ORDER BY, JOIN conditions on indexed columns at the cost of slightly slower writes.
```sql
CREATE INDEX idx_employees_salary ON employees(salary);
```

---

### Q64. Drop an index
**What it does:** Removes the salary index.
**Key concept:** `DROP INDEX` — safe to drop unused indexes; they only add write overhead. Check `pg_stat_user_indexes.idx_scan = 0` for unused indexes.
```sql
DROP INDEX idx_employees_salary;
```

---

### Q65. Unique index
**What it does:** Creates an index that also enforces uniqueness on email.
**Key concept:** A UNIQUE index does two things: speeds up lookups and prevents duplicate values. Equivalent to a UNIQUE constraint, but more explicit.
```sql
CREATE UNIQUE INDEX idx_employees_email ON employees(email);
```

---

### Q66. Truncate a table
**What it does:** Removes all rows from orders instantly.
**Key concept:** `TRUNCATE` is faster than `DELETE` (no row-by-row logging) but cannot be rolled back in older PostgreSQL. It also resets sequences if `RESTART IDENTITY` is added. Fires no row-level triggers.
```sql
TRUNCATE TABLE orders;
```

---

### Q67. Drop a table
**What it does:** Completely removes the orders table and its data.
**Key concept:** `DROP TABLE IF EXISTS` — prevents error if table doesn't exist. Add `CASCADE` to also drop foreign keys and dependent views.
```sql
DROP TABLE IF EXISTS orders;
```

---

### Q68. Create a view
**What it does:** Creates a saved query that behaves like a virtual table.
**Key concept:** A view is a named SELECT query stored in the catalog. It executes fresh every time you query it. No data is stored separately. Simplifies complex queries for downstream users.
```sql
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE is_active = TRUE;
```

---

### Q69. Query a view
**What it does:** Reads from the view exactly like a table.
**Key concept:** When you query a view, PostgreSQL expands it to its underlying SELECT. Performance depends on the underlying query and whether indexes are used.
```sql
SELECT * FROM active_employees;
```

---

### Q70. Drop a view
**What it does:** Removes the view definition (not the underlying data).
**Key concept:** `DROP VIEW IF EXISTS` — the original table is unaffected. Use `CASCADE` to drop dependent objects like other views built on top of this one.
```sql
DROP VIEW IF EXISTS active_employees;
```

---

### Q71. COALESCE — replace NULL with default
**What it does:** Returns 'Unassigned' if department is NULL, otherwise returns the department.
**Key concept:** `COALESCE(a, b, c, ...)` — returns the first non-NULL value in the list. Frequently used in SELECT and ORDER BY to handle optional columns.
```sql
SELECT name, COALESCE(department, 'Unassigned') AS dept FROM employees;
```

---

### Q72. NULLIF — return NULL when values match
**What it does:** Returns NULL if department equals 'Unknown'; otherwise returns department.
**Key concept:** `NULLIF(a, b)` returns NULL when `a = b`. Used to prevent division-by-zero: `amount / NULLIF(quantity, 0)`.
```sql
SELECT NULLIF(department, 'Unknown') AS dept FROM employees;
```

---

### Q73. CASE expression — salary bands
**What it does:** Labels each employee as 'Low', 'Medium', or 'High' based on salary.
**Key concept:** `CASE WHEN ... THEN ... ELSE ... END` — PostgreSQL's if-else in SQL. Can be used in SELECT, WHERE, ORDER BY, and GROUP BY.
```sql
SELECT name, salary,
  CASE
    WHEN salary < 40000 THEN 'Low'
    WHEN salary BETWEEN 40000 AND 80000 THEN 'Medium'
    ELSE 'High'
  END AS salary_band
FROM employees;
```

---

### Q74. Days between two dates
**What it does:** Subtracts two dates and returns the number of days.
**Key concept:** Date arithmetic in PostgreSQL: `DATE - DATE = INTEGER` (days). For intervals between timestamps use `EXTRACT(EPOCH FROM ts1 - ts2)`.
```sql
SELECT '2024-12-31'::DATE - '2024-01-01'::DATE AS days_diff;
```

---

### Q75. Add days to a date
**What it does:** Adds 30 days to each employee's hire_date.
**Key concept:** `date + INTERVAL '...'` — INTERVAL literals support: `'30 days'`, `'1 month'`, `'2 hours 30 minutes'`, `'1 year 3 months'`.
```sql
SELECT name, hire_date + INTERVAL '30 days' AS review_date FROM employees;
```

---

### Q76. First day of current month
**What it does:** Returns midnight on the 1st of the current month.
**Key concept:** `DATE_TRUNC(field, timestamp)` — truncates to the start of the given unit: 'year', 'month', 'week', 'day', 'hour', 'minute'. Essential for grouping time-series data.
```sql
SELECT DATE_TRUNC('month', CURRENT_DATE) AS first_of_month;
```

---

### Q77. Day of week name
**What it does:** Shows "Monday", "Tuesday", etc. for hire dates.
**Key concept:** `TO_CHAR(date, format)` — format codes: `'Day'` (full name padded), `'DY'` (3-letter abbr), `'D'` (numeric 1-7), `'YYYY-MM-DD'`, `'HH24:MI:SS'`.
```sql
SELECT name, TO_CHAR(hire_date, 'Day') AS day_of_week FROM employees;
```

---

### Q78. Format date as DD-MM-YYYY
**What it does:** Converts the stored date to a human-friendly Indian date format.
**Key concept:** `TO_CHAR` for custom output formatting. The stored type remains DATE — only the display changes.
```sql
SELECT name, TO_CHAR(hire_date, 'DD-MM-YYYY') AS formatted_date FROM employees;
```

---

### Q79. Cast NUMERIC to TEXT
**What it does:** Converts salary (a number) to a string.
**Key concept:** `value::TYPE` is PostgreSQL's cast syntax. Equivalent to `CAST(value AS TYPE)`. Needed when concatenating numbers with strings.
```sql
SELECT name, salary::TEXT AS salary_text FROM employees;
```

---

### Q80. Cast TEXT to INTEGER
**What it does:** Converts the string '42' to an integer and adds 8.
**Key concept:** Explicit cast. Will throw an error if the string is not a valid number. Use `NULLIF` + TRY approach or validate beforehand.
```sql
SELECT '42'::INTEGER + 8 AS result;
```

---

### Q81. BETWEEN with dates
**What it does:** Finds employees hired within calendar year 2022.
**Key concept:** Date BETWEEN is inclusive. `'2022-01-01'` to `'2022-12-31'` catches all days in 2022. For timestamps, use `< '2023-01-01'` to avoid missing Dec 31st time values.
```sql
SELECT * FROM employees
WHERE hire_date BETWEEN '2022-01-01' AND '2022-12-31';
```

---

### Q82. Case-insensitive LIKE
**What it does:** Finds employees whose name contains "kumar" in any case.
**Key concept:** `ILIKE` — case-insensitive version of LIKE. PostgreSQL-specific. For portability use `LIKE LOWER(pattern)`.
```sql
SELECT * FROM employees WHERE name ILIKE '%kumar%';
```

---

### Q83. Delete inactive employees
**What it does:** Removes all rows where is_active is false.
**Key concept:** Bulk DELETE with a condition. For very large deletes, consider batching to avoid long lock hold times.
```sql
DELETE FROM employees WHERE is_active = FALSE;
```

---

### Q84. Bulk update NULL departments
**What it does:** Sets department to 'General' for anyone without one.
**Key concept:** UPDATE with IS NULL condition — common data remediation pattern after importing data from external sources.
```sql
UPDATE employees SET department = 'General' WHERE department IS NULL;
```

---

### Q85. Calculated column — salary with bonus
**What it does:** Adds 10% to the salary for display.
**Key concept:** Arithmetic in SELECT — does not modify the stored value. To persist the calculation, use a GENERATED ALWAYS column or UPDATE.
```sql
SELECT name, salary, salary * 1.10 AS salary_with_bonus FROM employees;
```

---

### Q86. Extract email domain
**What it does:** Returns the part of the email after the @ symbol.
**Key concept:** `POSITION('@' IN email)` returns the index of @. `SUBSTRING(str FROM start)` returns everything from that position onward.
```sql
SELECT email, SUBSTRING(email FROM POSITION('@' IN email) + 1) AS domain
FROM employees;
```

---

### Q87. Every Nth row
**What it does:** Returns rows where the ID is divisible by 3.
**Key concept:** Modulo operator `%` — `id % 3 = 0` means "id is a multiple of 3." Useful for systematic sampling.
```sql
SELECT * FROM employees WHERE id % 3 = 0;
```

---

### Q88. Count NULLs in a column
**What it does:** Counts how many employees have no department assigned.
**Key concept:** Combining WHERE IS NULL with COUNT(*) — a data quality check. `COUNT(department)` alone would skip NULLs.
```sql
SELECT COUNT(*) AS null_dept_count
FROM employees WHERE department IS NULL;
```

---

### Q89. Total sales amount
**What it does:** Sums all revenue from the sales table.
**Key concept:** SUM on a fact table column — a basic KPI query. Returns NULL if table is empty (use `COALESCE(SUM(amount), 0)` to return 0 instead).
```sql
SELECT SUM(amount) AS total_sales FROM sales;
```

---

### Q90. Most recent order date
**What it does:** Finds when the latest order was placed.
**Key concept:** `MAX(date_column)` finds the latest/most recent date. `MIN(date_column)` finds the earliest.
```sql
SELECT MAX(order_date) AS latest_order FROM orders;
```

---

### Q91. Orders by status
**What it does:** Shows how many orders exist in each status (pending, shipped, etc.).
**Key concept:** GROUP BY on an enum-like column — produces a frequency distribution. Useful for operational dashboards.
```sql
SELECT status, COUNT(*) AS count
FROM orders
GROUP BY status;
```

---

### Q92. Second highest salary
**What it does:** Returns the salary just below the maximum.
**Key concept:** Nested `MAX` with a subquery exclusion. A classic interview question. For the Nth highest, use `LIMIT 1 OFFSET N-1` with DISTINCT ORDER BY.
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

---

### Q93. Products above average price
**What it does:** Filters products whose price exceeds the overall average.
**Key concept:** Scalar subquery in WHERE — the inner SELECT returns one value which is used in the outer comparison.
```sql
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

---

### Q94. Check if a table exists
**What it does:** Returns TRUE/FALSE depending on whether the table is present.
**Key concept:** `information_schema.tables` — PostgreSQL's catalog of all tables. Used in migrations and setup scripts to avoid errors.
```sql
SELECT EXISTS (
    SELECT FROM information_schema.tables
    WHERE table_name = 'employees'
);
```

---

### Q95. List all columns of a table
**What it does:** Describes the schema of the employees table.
**Key concept:** `information_schema.columns` — standard SQL catalog. Alternatives: `\d employees` in psql, or query `pg_attribute` for lower-level details.
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'employees';
```

---

### Q96. Character at a specific position
**What it does:** Returns the character at position 3 of the name.
**Key concept:** `SUBSTRING(str, start, length)` — 1-indexed in SQL (not 0-indexed like programming languages). Returns 1 character starting at position 3.
```sql
SELECT name, SUBSTRING(name, 3, 1) AS third_char FROM employees;
```

---

### Q97. Repeat a string
**What it does:** Returns the string '*' repeated 5 times as '*****'.
**Key concept:** `REPEAT(text, n)` — useful for formatting, generating test data, or building separator strings.
```sql
SELECT REPEAT('*', 5) AS stars;
```

---

### Q98. Reverse a string
**What it does:** Returns each name spelled backwards.
**Key concept:** `REVERSE(text)` — PostgreSQL built-in. Used in palindrome checks and certain pattern matching scenarios.
```sql
SELECT name, REVERSE(name) AS reversed_name FROM employees;
```

---

### Q99. Employees hired per year
**What it does:** Shows hiring volume broken down by year.
**Key concept:** EXTRACT + GROUP BY pattern — converts dates into groupable year values. Ordered chronologically.
```sql
SELECT EXTRACT(YEAR FROM hire_date) AS year, COUNT(*) AS hired
FROM employees
GROUP BY year
ORDER BY year;
```

---

### Q100. Sort by multiple columns
**What it does:** Groups employees by department alphabetically, then sorts by salary within each department.
**Key concept:** Multi-column ORDER BY — defines a sort hierarchy. The second column only breaks ties in the first.
```sql
SELECT * FROM employees
ORDER BY department ASC, salary DESC;
```
