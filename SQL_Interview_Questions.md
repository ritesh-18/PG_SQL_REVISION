# 300 PostgreSQL SQL Interview Questions

> All queries are written in **PostgreSQL** syntax.
> Tables used throughout this file are defined in the schema below.

---

## Reference Schema

```sql
-- Core tables used in examples
CREATE TABLE departments (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    location    VARCHAR(100),
    budget      NUMERIC(15,2)
);

CREATE TABLE employees (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    department  VARCHAR(100),
    department_id INT REFERENCES departments(id),
    salary      NUMERIC(12,2),
    manager_id  INT REFERENCES employees(id),
    hire_date   DATE,
    email       VARCHAR(150),
    is_active   BOOLEAN DEFAULT TRUE
);

CREATE TABLE customers (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(100) NOT NULL,
    email       VARCHAR(150),
    phone       VARCHAR(15),
    city        VARCHAR(100),
    country     VARCHAR(100),
    created_at  TIMESTAMP DEFAULT NOW()
);

CREATE TABLE products (
    id          SERIAL PRIMARY KEY,
    name        VARCHAR(150) NOT NULL,
    category    VARCHAR(100),
    price       NUMERIC(10,2),
    stock_qty   INT DEFAULT 0,
    description TEXT,
    metadata    JSONB
);

CREATE TABLE orders (
    id          SERIAL PRIMARY KEY,
    customer_id INT REFERENCES customers(id),
    product_id  INT REFERENCES products(id),
    quantity    INT,
    amount      NUMERIC(12,2),
    order_date  DATE DEFAULT CURRENT_DATE,
    status      VARCHAR(50) DEFAULT 'pending'
);

CREATE TABLE sales (
    id            SERIAL PRIMARY KEY,
    salesperson_id INT REFERENCES employees(id),
    region        VARCHAR(100),
    amount        NUMERIC(12,2),
    sale_date     DATE
);
```

---

## EASY — Questions 1 to 100

---

### Q1. Select all columns from employees
```sql
SELECT * FROM employees;
```

---

### Q2. Select only name and salary from employees
```sql
SELECT name, salary FROM employees;
```

---

### Q3. Select employees with salary greater than 50000
```sql
SELECT * FROM employees WHERE salary > 50000;
```

---

### Q4. Select employees in the 'Engineering' department
```sql
SELECT * FROM employees WHERE department = 'Engineering';
```

---

### Q5. Select top 5 highest-paid employees
```sql
SELECT * FROM employees ORDER BY salary DESC LIMIT 5;
```

---

### Q6. Count total number of employees
```sql
SELECT COUNT(*) AS total_employees FROM employees;
```

---

### Q7. Find the maximum salary
```sql
SELECT MAX(salary) AS max_salary FROM employees;
```

---

### Q8. Find the minimum salary
```sql
SELECT MIN(salary) AS min_salary FROM employees;
```

---

### Q9. Find the average salary
```sql
SELECT ROUND(AVG(salary), 2) AS avg_salary FROM employees;
```

---

### Q10. Find the sum of all salaries
```sql
SELECT SUM(salary) AS total_salary FROM employees;
```

---

### Q11. Count employees per department
```sql
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department;
```

---

### Q12. Find departments with more than 5 employees
```sql
SELECT department, COUNT(*) AS total
FROM employees
GROUP BY department
HAVING COUNT(*) > 5;
```

---

### Q13. Select distinct departments
```sql
SELECT DISTINCT department FROM employees;
```

---

### Q14. Select employees hired after 2020-01-01
```sql
SELECT * FROM employees WHERE hire_date > '2020-01-01';
```

---

### Q15. Select employees whose name starts with 'R'
```sql
SELECT * FROM employees WHERE name LIKE 'R%';
```

---

### Q16. Select employees whose name ends with 'a'
```sql
SELECT * FROM employees WHERE name LIKE '%a';
```

---

### Q17. Select employees with salary between 40000 and 80000
```sql
SELECT * FROM employees WHERE salary BETWEEN 40000 AND 80000;
```

---

### Q18. Select employees in HR or Finance department
```sql
SELECT * FROM employees WHERE department IN ('HR', 'Finance');
```

---

### Q19. Select employees where manager_id is NULL (no manager)
```sql
SELECT * FROM employees WHERE manager_id IS NULL;
```

---

### Q20. Select employees where email is NOT NULL
```sql
SELECT * FROM employees WHERE email IS NOT NULL;
```

---

### Q21. Sort employees by hire_date ascending
```sql
SELECT * FROM employees ORDER BY hire_date ASC;
```

---

### Q22. Sort employees by salary descending, name ascending
```sql
SELECT * FROM employees ORDER BY salary DESC, name ASC;
```

---

### Q23. Insert a new employee record
```sql
INSERT INTO employees (name, department, salary, hire_date, email)
VALUES ('Priya Sharma', 'Engineering', 75000, '2024-01-15', 'priya@example.com');
```

---

### Q24. Update salary of employee with id = 5
```sql
UPDATE employees SET salary = 90000 WHERE id = 5;
```

---

### Q25. Delete an employee with id = 10
```sql
DELETE FROM employees WHERE id = 10;
```

---

### Q26. Select employees and alias salary as monthly_salary
```sql
SELECT name, salary AS monthly_salary FROM employees;
```

---

### Q27. Find employees not in Engineering department
```sql
SELECT * FROM employees WHERE department != 'Engineering';
-- OR
SELECT * FROM employees WHERE department <> 'Engineering';
```

---

### Q28. Select employees with NULL department
```sql
SELECT * FROM employees WHERE department IS NULL;
```

---

### Q29. Convert employee name to uppercase
```sql
SELECT UPPER(name) AS upper_name FROM employees;
```

---

### Q30. Convert employee name to lowercase
```sql
SELECT LOWER(name) AS lower_name FROM employees;
```

---

### Q31. Get length of each employee name
```sql
SELECT name, LENGTH(name) AS name_length FROM employees;
```

---

### Q32. Concatenate first and last name (simulated with name + department)
```sql
SELECT name || ' - ' || department AS employee_info FROM employees;
```

---

### Q33. Trim spaces from employee name
```sql
SELECT TRIM(name) AS trimmed_name FROM employees;
```

---

### Q34. Get current date
```sql
SELECT CURRENT_DATE;
```

---

### Q35. Get current timestamp
```sql
SELECT NOW();
-- OR
SELECT CURRENT_TIMESTAMP;
```

---

### Q36. Get year from hire_date
```sql
SELECT name, EXTRACT(YEAR FROM hire_date) AS hire_year FROM employees;
```

---

### Q37. Get month from hire_date
```sql
SELECT name, EXTRACT(MONTH FROM hire_date) AS hire_month FROM employees;
```

---

### Q38. Calculate age of an employee based on hire_date
```sql
SELECT name, AGE(NOW(), hire_date) AS tenure FROM employees;
```

---

### Q39. Select employees hired in the year 2022
```sql
SELECT * FROM employees WHERE EXTRACT(YEAR FROM hire_date) = 2022;
```

---

### Q40. Get first 3 characters of employee name
```sql
SELECT name, LEFT(name, 3) AS short_name FROM employees;
```

---

### Q41. Get last 3 characters of employee name
```sql
SELECT name, RIGHT(name, 3) AS suffix FROM employees;
```

---

### Q42. Replace 'Engineer' with 'Dev' in department column
```sql
SELECT name, REPLACE(department, 'Engineer', 'Dev') AS dept FROM employees;
```

---

### Q43. Round salary to 0 decimal places
```sql
SELECT name, ROUND(salary, 0) AS rounded_salary FROM employees;
```

---

### Q44. Get absolute value of a number
```sql
SELECT ABS(-5000);
```

---

### Q45. Get ceiling and floor of salary
```sql
SELECT name, CEIL(salary) AS ceil_sal, FLOOR(salary) AS floor_sal FROM employees;
```

---

### Q46. Select employees using LIMIT and OFFSET (pagination)
```sql
-- Page 2, 10 records per page
SELECT * FROM employees ORDER BY id LIMIT 10 OFFSET 10;
```

---

### Q47. Count number of distinct departments
```sql
SELECT COUNT(DISTINCT department) AS dept_count FROM employees;
```

---

### Q48. Get average salary per department
```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department;
```

---

### Q49. Find departments where average salary > 60000
```sql
SELECT department, ROUND(AVG(salary), 2) AS avg_salary
FROM employees
GROUP BY department
HAVING AVG(salary) > 60000;
```

---

### Q50. Select employees with salary NOT between 30000 and 50000
```sql
SELECT * FROM employees WHERE salary NOT BETWEEN 30000 AND 50000;
```

---

### Q51. Find total orders per customer
```sql
SELECT customer_id, COUNT(*) AS total_orders
FROM orders
GROUP BY customer_id;
```

---

### Q52. Find total order amount per customer
```sql
SELECT customer_id, SUM(amount) AS total_spent
FROM orders
GROUP BY customer_id;
```

---

### Q53. Select all customers from India
```sql
SELECT * FROM customers WHERE country = 'India';
```

---

### Q54. Find customers who have not placed any orders (simple subquery)
```sql
SELECT * FROM customers
WHERE id NOT IN (SELECT DISTINCT customer_id FROM orders);
```

---

### Q55. Get all products with price less than 500
```sql
SELECT * FROM products WHERE price < 500;
```

---

### Q56. Get products where stock is 0 (out of stock)
```sql
SELECT * FROM products WHERE stock_qty = 0;
```

---

### Q57. Get max price per product category
```sql
SELECT category, MAX(price) AS max_price
FROM products
GROUP BY category;
```

---

### Q58. Count products per category
```sql
SELECT category, COUNT(*) AS product_count
FROM products
GROUP BY category;
```

---

### Q59. Add a new column to employees table
```sql
ALTER TABLE employees ADD COLUMN phone VARCHAR(15);
```

---

### Q60. Drop a column from employees table
```sql
ALTER TABLE employees DROP COLUMN phone;
```

---

### Q61. Rename a column
```sql
ALTER TABLE employees RENAME COLUMN is_active TO active_status;
```

---

### Q62. Add NOT NULL constraint to an existing column
```sql
ALTER TABLE employees ALTER COLUMN email SET NOT NULL;
```

---

### Q63. Create an index on salary column
```sql
CREATE INDEX idx_employees_salary ON employees(salary);
```

---

### Q64. Drop an index
```sql
DROP INDEX idx_employees_salary;
```

---

### Q65. Create a unique index on email
```sql
CREATE UNIQUE INDEX idx_employees_email ON employees(email);
```

---

### Q66. Truncate a table (remove all rows, keep structure)
```sql
TRUNCATE TABLE orders;
```

---

### Q67. Drop a table
```sql
DROP TABLE IF EXISTS orders;
```

---

### Q68. Create a simple view
```sql
CREATE VIEW active_employees AS
SELECT * FROM employees WHERE is_active = TRUE;
```

---

### Q69. Select from a view
```sql
SELECT * FROM active_employees;
```

---

### Q70. Drop a view
```sql
DROP VIEW IF EXISTS active_employees;
```

---

### Q71. Use COALESCE to replace NULL with a default value
```sql
SELECT name, COALESCE(department, 'Unassigned') AS dept FROM employees;
```

---

### Q72. Use NULLIF to return NULL when two values are equal
```sql
SELECT NULLIF(department, 'Unknown') AS dept FROM employees;
```

---

### Q73. Use CASE for salary bands
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

### Q74. Get number of days between two dates
```sql
SELECT '2024-12-31'::DATE - '2024-01-01'::DATE AS days_diff;
```

---

### Q75. Add 30 days to hire_date
```sql
SELECT name, hire_date + INTERVAL '30 days' AS review_date FROM employees;
```

---

### Q76. Get the first day of the current month
```sql
SELECT DATE_TRUNC('month', CURRENT_DATE) AS first_of_month;
```

---

### Q77. Get day of week for hire_date
```sql
SELECT name, TO_CHAR(hire_date, 'Day') AS day_of_week FROM employees;
```

---

### Q78. Format date as DD-MM-YYYY
```sql
SELECT name, TO_CHAR(hire_date, 'DD-MM-YYYY') AS formatted_date FROM employees;
```

---

### Q79. Cast salary from NUMERIC to TEXT
```sql
SELECT name, salary::TEXT AS salary_text FROM employees;
```

---

### Q80. Cast a string to integer
```sql
SELECT '42'::INTEGER + 8 AS result;
```

---

### Q81. Select employees using BETWEEN for hire_date
```sql
SELECT * FROM employees
WHERE hire_date BETWEEN '2022-01-01' AND '2022-12-31';
```

---

### Q82. Find employees whose name contains 'Kumar'
```sql
SELECT * FROM employees WHERE name ILIKE '%kumar%';
```

---

### Q83. Delete all inactive employees
```sql
DELETE FROM employees WHERE is_active = FALSE;
```

---

### Q84. Update department for all employees without one
```sql
UPDATE employees SET department = 'General' WHERE department IS NULL;
```

---

### Q85. Select employees and show salary + 10% bonus
```sql
SELECT name, salary, salary * 1.10 AS salary_with_bonus FROM employees;
```

---

### Q86. Get substring from email (domain part)
```sql
SELECT email, SUBSTRING(email FROM POSITION('@' IN email) + 1) AS domain
FROM employees;
```

---

### Q87. Select every Nth row (e.g., every 3rd)
```sql
SELECT * FROM employees WHERE id % 3 = 0;
```

---

### Q88. Count NULL values in department column
```sql
SELECT COUNT(*) AS null_dept_count
FROM employees WHERE department IS NULL;
```

---

### Q89. Get total sales amount
```sql
SELECT SUM(amount) AS total_sales FROM sales;
```

---

### Q90. Find the most recent order date
```sql
SELECT MAX(order_date) AS latest_order FROM orders;
```

---

### Q91. Count orders by status
```sql
SELECT status, COUNT(*) AS count
FROM orders
GROUP BY status;
```

---

### Q92. Select the 2nd highest salary (simple approach)
```sql
SELECT MAX(salary) AS second_highest
FROM employees
WHERE salary < (SELECT MAX(salary) FROM employees);
```

---

### Q93. Get products with price above average price
```sql
SELECT * FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

---

### Q94. Check if a table exists
```sql
SELECT EXISTS (
    SELECT FROM information_schema.tables
    WHERE table_name = 'employees'
);
```

---

### Q95. List all columns of a table
```sql
SELECT column_name, data_type
FROM information_schema.columns
WHERE table_name = 'employees';
```

---

### Q96. Get the character at position 3 of name
```sql
SELECT name, SUBSTRING(name, 3, 1) AS third_char FROM employees;
```

---

### Q97. Repeat a string N times
```sql
SELECT REPEAT('*', 5) AS stars;
```

---

### Q98. Reverse a string
```sql
SELECT name, REVERSE(name) AS reversed_name FROM employees;
```

---

### Q99. Count employees hired per year
```sql
SELECT EXTRACT(YEAR FROM hire_date) AS year, COUNT(*) AS hired
FROM employees
GROUP BY year
ORDER BY year;
```

---

### Q100. Select employees ordered by department then by salary descending
```sql
SELECT * FROM employees
ORDER BY department ASC, salary DESC;
```

---

## MEDIUM — Questions 101 to 200

---

### Q101. INNER JOIN employees with departments
```sql
SELECT e.name, e.salary, d.name AS department_name, d.location
FROM employees e
INNER JOIN departments d ON e.department_id = d.id;
```

---

### Q102. LEFT JOIN — all employees, even without a department
```sql
SELECT e.name, d.name AS department_name
FROM employees e
LEFT JOIN departments d ON e.department_id = d.id;
```

---

### Q103. RIGHT JOIN — all departments, even without employees
```sql
SELECT e.name, d.name AS department_name
FROM employees e
RIGHT JOIN departments d ON e.department_id = d.id;
```

---

### Q104. FULL OUTER JOIN — all employees and all departments
```sql
SELECT e.name, d.name AS department_name
FROM employees e
FULL OUTER JOIN departments d ON e.department_id = d.id;
```

---

### Q105. CROSS JOIN — every employee with every department
```sql
SELECT e.name AS employee, d.name AS department
FROM employees e
CROSS JOIN departments d;
```

---

### Q106. Self JOIN — find employee and their manager
```sql
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON e.manager_id = m.id;
```

---

### Q107. Find customers with their total order count and amount
```sql
SELECT c.name, COUNT(o.id) AS orders, SUM(o.amount) AS total_spent
FROM customers c
LEFT JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name;
```

---

### Q108. Find customers who have placed more than 3 orders
```sql
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON c.id = o.customer_id
GROUP BY c.id, c.name
HAVING COUNT(o.id) > 3;
```

---

### Q109. Find the top-selling product by total quantity
```sql
SELECT p.name, SUM(o.quantity) AS total_qty
FROM products p
JOIN orders o ON p.id = o.product_id
GROUP BY p.id, p.name
ORDER BY total_qty DESC
LIMIT 1;
```

---

### Q110. UNION — combine employees from two departments
```sql
SELECT name, salary FROM employees WHERE department = 'Engineering'
UNION
SELECT name, salary FROM employees WHERE department = 'Finance';
```

---

### Q111. UNION ALL — include duplicates
```sql
SELECT department FROM employees WHERE salary > 60000
UNION ALL
SELECT department FROM employees WHERE hire_date > '2023-01-01';
```

---

### Q112. INTERSECT — employees in both conditions
```sql
SELECT name FROM employees WHERE salary > 50000
INTERSECT
SELECT name FROM employees WHERE department = 'Engineering';
```

---

### Q113. EXCEPT — employees in first set but not in second
```sql
SELECT name FROM employees WHERE salary > 50000
EXCEPT
SELECT name FROM employees WHERE department = 'HR';
```

---

### Q114. Subquery — employees earning above department average
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

### Q115. EXISTS — customers who have placed at least one order
```sql
SELECT * FROM customers c
WHERE EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

---

### Q116. NOT EXISTS — customers with no orders
```sql
SELECT * FROM customers c
WHERE NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = c.id
);
```

---

### Q117. IN with subquery — employees in departments with budget > 1 million
```sql
SELECT * FROM employees
WHERE department_id IN (
    SELECT id FROM departments WHERE budget > 1000000
);
```

---

### Q118. CTE — basic Common Table Expression
```sql
WITH high_earners AS (
    SELECT * FROM employees WHERE salary > 80000
)
SELECT * FROM high_earners ORDER BY salary DESC;
```

---

### Q119. CTE — calculate department average and compare
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

### Q120. ROW_NUMBER — assign row numbers per department ordered by salary
```sql
SELECT name, department, salary,
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
FROM employees;
```

---

### Q121. RANK — rank employees by salary within department
```sql
SELECT name, department, salary,
  RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS rnk
FROM employees;
```

---

### Q122. DENSE_RANK — dense ranking by salary (no gaps)
```sql
SELECT name, department, salary,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rnk
FROM employees;
```

---

### Q123. Top 1 employee per department using ROW_NUMBER
```sql
WITH ranked AS (
    SELECT *,
      ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS rn
    FROM employees
)
SELECT * FROM ranked WHERE rn = 1;
```

---

### Q124. LAG — compare current salary with previous employee's salary
```sql
SELECT name, salary,
  LAG(salary) OVER (ORDER BY salary) AS prev_salary,
  salary - LAG(salary) OVER (ORDER BY salary) AS diff
FROM employees;
```

---

### Q125. LEAD — compare current salary with next employee's salary
```sql
SELECT name, salary,
  LEAD(salary) OVER (ORDER BY salary) AS next_salary
FROM employees;
```

---

### Q126. SUM as running total (cumulative sum)
```sql
SELECT name, hire_date, salary,
  SUM(salary) OVER (ORDER BY hire_date) AS running_total
FROM employees;
```

---

### Q127. Running average salary ordered by hire_date
```sql
SELECT name, hire_date, salary,
  ROUND(AVG(salary) OVER (ORDER BY hire_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW), 2) AS running_avg
FROM employees;
```

---

### Q128. FIRST_VALUE — get highest salary in each department alongside every row
```sql
SELECT name, department, salary,
  FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS highest_in_dept
FROM employees;
```

---

### Q129. LAST_VALUE — lowest salary in department (needs frame)
```sql
SELECT name, department, salary,
  LAST_VALUE(salary) OVER (
    PARTITION BY department ORDER BY salary DESC
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
  ) AS lowest_in_dept
FROM employees;
```

---

### Q130. NTILE — divide employees into 4 salary quartiles
```sql
SELECT name, salary,
  NTILE(4) OVER (ORDER BY salary) AS quartile
FROM employees;
```

---

### Q131. PERCENT_RANK — relative rank as percentage
```sql
SELECT name, salary,
  ROUND(PERCENT_RANK() OVER (ORDER BY salary)::NUMERIC, 4) AS pct_rank
FROM employees;
```

---

### Q132. CUME_DIST — cumulative distribution of salary
```sql
SELECT name, salary,
  ROUND(CUME_DIST() OVER (ORDER BY salary)::NUMERIC, 4) AS cume_dist
FROM employees;
```

---

### Q133. Multiple CTEs chained
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

### Q134. CASE with aggregation — count employees per salary band
```sql
SELECT
  SUM(CASE WHEN salary < 40000 THEN 1 ELSE 0 END) AS low,
  SUM(CASE WHEN salary BETWEEN 40000 AND 80000 THEN 1 ELSE 0 END) AS medium,
  SUM(CASE WHEN salary > 80000 THEN 1 ELSE 0 END) AS high
FROM employees;
```

---

### Q135. Conditional aggregation with FILTER
```sql
SELECT
  COUNT(*) FILTER (WHERE department = 'Engineering') AS eng_count,
  COUNT(*) FILTER (WHERE salary > 70000) AS high_earners,
  AVG(salary) FILTER (WHERE is_active = TRUE) AS active_avg_sal
FROM employees;
```

---

### Q136. Find duplicate email addresses
```sql
SELECT email, COUNT(*) AS cnt
FROM employees
WHERE email IS NOT NULL
GROUP BY email
HAVING COUNT(*) > 1;
```

---

### Q137. Delete duplicate rows keeping only the first
```sql
DELETE FROM employees
WHERE id NOT IN (
    SELECT MIN(id)
    FROM employees
    GROUP BY email
);
```

---

### Q138. UPSERT — INSERT or UPDATE with ON CONFLICT
```sql
INSERT INTO employees (id, name, salary, department)
VALUES (1, 'Rahul Verma', 85000, 'Engineering')
ON CONFLICT (id)
DO UPDATE SET salary = EXCLUDED.salary, name = EXCLUDED.name;
```

---

### Q139. ON CONFLICT DO NOTHING
```sql
INSERT INTO employees (id, name, salary)
VALUES (1, 'Anita Rao', 70000)
ON CONFLICT (id) DO NOTHING;
```

---

### Q140. INSERT from SELECT
```sql
INSERT INTO employees (name, department, salary)
SELECT name, 'General', salary * 0.9
FROM employees
WHERE is_active = FALSE;
```

---

### Q141. UPDATE using JOIN (UPDATE with FROM)
```sql
UPDATE employees e
SET salary = e.salary * 1.10
FROM departments d
WHERE e.department_id = d.id
  AND d.name = 'Engineering';
```

---

### Q142. String aggregation — list employees per department
```sql
SELECT department, STRING_AGG(name, ', ' ORDER BY name) AS emp_list
FROM employees
GROUP BY department;
```

---

### Q143. Array aggregation
```sql
SELECT department, ARRAY_AGG(name ORDER BY salary DESC) AS employees
FROM employees
GROUP BY department;
```

---

### Q144. Unnest an array
```sql
SELECT UNNEST(ARRAY['Rahul', 'Priya', 'Amit']) AS name;
```

---

### Q145. Generate a series of dates
```sql
SELECT generate_series('2024-01-01'::DATE, '2024-12-31'::DATE, '1 month') AS month;
```

---

### Q146. Generate a series of numbers
```sql
SELECT generate_series(1, 10) AS num;
```

---

### Q147. Find Nth highest salary using subquery
```sql
-- 3rd highest salary
SELECT DISTINCT salary
FROM employees
ORDER BY salary DESC
LIMIT 1 OFFSET 2;
```

---

### Q148. Find gaps in employee IDs (missing IDs)
```sql
SELECT id + 1 AS missing_from
FROM employees e1
WHERE NOT EXISTS (
    SELECT 1 FROM employees e2 WHERE e2.id = e1.id + 1
)
AND id < (SELECT MAX(id) FROM employees);
```

---

### Q149. LATERAL join — get last 2 orders per customer
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

### Q150. Calculate year-over-year sales growth
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

### Q151. Moving average — 3-month rolling average of sales
```sql
SELECT sale_date, amount,
  ROUND(AVG(amount) OVER (ORDER BY sale_date ROWS BETWEEN 2 PRECEDING AND CURRENT ROW), 2) AS moving_avg_3
FROM sales;
```

---

### Q152. Pivot — sales by region across quarters (using FILTER)
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

### Q153. Find employees who earn more than their manager
```sql
SELECT e.name AS employee, e.salary AS emp_salary,
       m.name AS manager, m.salary AS mgr_salary
FROM employees e
JOIN employees m ON e.manager_id = m.id
WHERE e.salary > m.salary;
```

---

### Q154. COALESCE with multiple fallbacks
```sql
SELECT name,
  COALESCE(email, phone, 'No contact') AS contact_info
FROM employees;
```

---

### Q155. Find orders placed in the last 30 days
```sql
SELECT * FROM orders
WHERE order_date >= CURRENT_DATE - INTERVAL '30 days';
```

---

### Q156. Get orders with their customer name and product name
```sql
SELECT o.id, c.name AS customer, p.name AS product,
       o.quantity, o.amount, o.order_date
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN products p ON o.product_id = p.id;
```

---

### Q157. Find months with no sales (missing months from series)
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

### Q158. Find customers who ordered every product (relational division)
```sql
SELECT customer_id
FROM orders
GROUP BY customer_id
HAVING COUNT(DISTINCT product_id) = (SELECT COUNT(*) FROM products);
```

---

### Q159. Calculate percentage of total sales per region
```sql
SELECT region,
  SUM(amount) AS region_sales,
  ROUND(SUM(amount) * 100.0 / SUM(SUM(amount)) OVER (), 2) AS pct_of_total
FROM sales
GROUP BY region;
```

---

### Q160. WINDOW clause — reuse window definition
```sql
SELECT name, salary,
  RANK()        OVER w AS rnk,
  DENSE_RANK()  OVER w AS dense_rnk,
  ROW_NUMBER()  OVER w AS rn
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);
```

---

### Q161. Find the most recent sale per salesperson
```sql
SELECT DISTINCT ON (salesperson_id)
  salesperson_id, sale_date, amount
FROM sales
ORDER BY salesperson_id, sale_date DESC;
```

---

### Q162. Update multiple columns at once
```sql
UPDATE employees
SET salary = salary * 1.05,
    department = 'Senior ' || department
WHERE hire_date < '2020-01-01';
```

---

### Q163. Recursive CTE — basic number series
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
```sql
-- Both return the same result
SELECT EXTRACT(YEAR FROM NOW()),
       DATE_PART('year', NOW());
```

---

### Q165. TO_TIMESTAMP — convert string to timestamp
```sql
SELECT TO_TIMESTAMP('2024-06-15 10:30:00', 'YYYY-MM-DD HH24:MI:SS');
```

---

### Q166. DATE_TRUNC to group by week
```sql
SELECT DATE_TRUNC('week', sale_date) AS week_start,
       SUM(amount) AS weekly_sales
FROM sales
GROUP BY week_start
ORDER BY week_start;
```

---

### Q167. BETWEEN with timestamps
```sql
SELECT * FROM orders
WHERE order_date BETWEEN '2024-01-01' AND '2024-06-30';
```

---

### Q168. Add a computed column in SELECT using arithmetic
```sql
SELECT name, salary,
  ROUND(salary * 12, 2) AS annual_salary,
  ROUND(salary * 0.20, 2) AS tax_estimate
FROM employees;
```

---

### Q169. Combine aggregate and non-aggregate using window function
```sql
SELECT name, department, salary,
  AVG(salary) OVER (PARTITION BY department) AS dept_avg,
  salary - AVG(salary) OVER (PARTITION BY department) AS diff_from_avg
FROM employees;
```

---

### Q170. Find employees with the same salary
```sql
SELECT e1.name, e2.name, e1.salary
FROM employees e1
JOIN employees e2 ON e1.salary = e2.salary AND e1.id < e2.id;
```

---

### Q171. Count orders by day of week
```sql
SELECT TO_CHAR(order_date, 'Day') AS day_name,
       COUNT(*) AS order_count
FROM orders
GROUP BY TO_CHAR(order_date, 'Day'), EXTRACT(DOW FROM order_date)
ORDER BY EXTRACT(DOW FROM order_date);
```

---

### Q172. Create a materialized view
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

### Q173. Refresh a materialized view
```sql
REFRESH MATERIALIZED VIEW dept_salary_stats;
-- Non-blocking version:
REFRESH MATERIALIZED VIEW CONCURRENTLY dept_salary_stats;
```

---

### Q174. Create a simple function
```sql
CREATE OR REPLACE FUNCTION get_employee_count(dept TEXT)
RETURNS INTEGER AS $$
    SELECT COUNT(*) FROM employees WHERE department = dept;
$$ LANGUAGE SQL;

-- Usage:
SELECT get_employee_count('Engineering');
```

---

### Q175. ILIKE — case-insensitive LIKE
```sql
SELECT * FROM customers WHERE name ILIKE '%sharma%';
```

---

### Q176. Regular expression match
```sql
-- Find emails with gmail
SELECT * FROM employees WHERE email ~ '@gmail\.com$';

-- Case insensitive regex
SELECT * FROM employees WHERE email ~* '@gmail\.com$';
```

---

### Q177. SIMILAR TO for pattern matching
```sql
SELECT * FROM employees WHERE name SIMILAR TO '(Rahul|Priya|Amit)%';
```

---

### Q178. ARRAY operations
```sql
-- Array contains element
SELECT * FROM products WHERE ARRAY['electronics', 'mobile'] && ARRAY['mobile'];

-- Array length
SELECT ARRAY_LENGTH(ARRAY[1,2,3,4], 1);
```

---

### Q179. String to array and back
```sql
SELECT STRING_TO_ARRAY('Rahul,Priya,Amit', ',') AS names;
SELECT ARRAY_TO_STRING(ARRAY['Rahul','Priya','Amit'], ' | ') AS name_list;
```

---

### Q180. Window function with ROWS BETWEEN
```sql
-- Sum of current and previous 2 rows
SELECT sale_date, amount,
  SUM(amount) OVER (
    ORDER BY sale_date
    ROWS BETWEEN 2 PRECEDING AND CURRENT ROW
  ) AS sliding_3day_sum
FROM sales;
```

---

### Q181. Find the median salary
```sql
SELECT PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees;
```

---

### Q182. PERCENTILE — 90th percentile salary
```sql
SELECT PERCENTILE_CONT(0.90) WITHIN GROUP (ORDER BY salary) AS p90_salary
FROM employees;
```

---

### Q183. MODE — most common salary
```sql
SELECT MODE() WITHIN GROUP (ORDER BY salary) AS most_common_salary
FROM employees;
```

---

### Q184. STDDEV and VARIANCE of salary
```sql
SELECT
  ROUND(STDDEV(salary)::NUMERIC, 2) AS std_dev,
  ROUND(VARIANCE(salary)::NUMERIC, 2) AS variance
FROM employees;
```

---

### Q185. Find orders where amount is above the 75th percentile
```sql
SELECT * FROM orders
WHERE amount > (
    SELECT PERCENTILE_CONT(0.75) WITHIN GROUP (ORDER BY amount)
    FROM orders
);
```

---

### Q186. Consecutive logins / consecutive dates (islands problem)
```sql
-- Find groups of consecutive order days per customer
WITH numbered AS (
    SELECT customer_id, order_date,
      order_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date)::INT AS grp
    FROM (SELECT DISTINCT customer_id, order_date FROM orders) t
)
SELECT customer_id, MIN(order_date), MAX(order_date),
       COUNT(*) AS consecutive_days
FROM numbered
GROUP BY customer_id, grp
HAVING COUNT(*) > 1;
```

---

### Q187. Calculate month-over-month change
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

### Q188. EXPLAIN — see query plan
```sql
EXPLAIN SELECT * FROM employees WHERE salary > 80000;
```

---

### Q189. EXPLAIN ANALYZE — run and profile query
```sql
EXPLAIN ANALYZE SELECT * FROM employees WHERE department = 'Engineering';
```

---

### Q190. Create composite index
```sql
CREATE INDEX idx_emp_dept_sal ON employees(department, salary DESC);
```

---

### Q191. Partial index — index only active employees
```sql
CREATE INDEX idx_active_emp ON employees(department) WHERE is_active = TRUE;
```

---

### Q192. Find tables without primary keys (catalog query)
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

### Q193. Get all indexes on a table
```sql
SELECT indexname, indexdef
FROM pg_indexes
WHERE tablename = 'employees';
```

---

### Q194. Check table size
```sql
SELECT pg_size_pretty(pg_total_relation_size('employees')) AS table_size;
```

---

### Q195. List all schemas in database
```sql
SELECT schema_name FROM information_schema.schemata;
```

---

### Q196. Create a schema
```sql
CREATE SCHEMA reports;
```

---

### Q197. Set search path
```sql
SET search_path TO reports, public;
```

---

### Q198. Rename a table
```sql
ALTER TABLE employees RENAME TO staff;
```

---

### Q199. Add a CHECK constraint
```sql
ALTER TABLE employees
ADD CONSTRAINT chk_salary_positive CHECK (salary >= 0);
```

---

### Q200. Create a table with all common constraints
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

---

## HARD — Questions 201 to 300

---

### Q201. Recursive CTE — employee hierarchy (all levels)
```sql
WITH RECURSIVE org_tree AS (
    -- Anchor: top-level employees (no manager)
    SELECT id, name, manager_id, 0 AS level, name::TEXT AS path
    FROM employees
    WHERE manager_id IS NULL

    UNION ALL

    -- Recursive: each employee's reports
    SELECT e.id, e.name, e.manager_id, t.level + 1,
           t.path || ' > ' || e.name
    FROM employees e
    JOIN org_tree t ON e.manager_id = t.id
)
SELECT id, name, level, path FROM org_tree ORDER BY path;
```

---

### Q202. Recursive CTE — subtree under a given manager
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
```sql
WITH RECURSIVE fib(a, b) AS (
    SELECT 0, 1
    UNION ALL
    SELECT b, a + b FROM fib WHERE b < 1000
)
SELECT a AS fibonacci FROM fib;
```

---

### Q204. Gaps and Islands — find consecutive date ranges
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

### Q205. Detect gaps in a sequence (missing IDs)
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

### Q206. Sessionize events — group events within 30-minute windows
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

### Q207. GROUPING SETS — multiple GROUP BY in one query
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
```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS yr,
       SUM(salary) AS total_salary
FROM employees
GROUP BY ROLLUP(department, EXTRACT(YEAR FROM hire_date))
ORDER BY department, yr;
```

---

### Q209. CUBE — all combinations of grouping
```sql
SELECT region, EXTRACT(YEAR FROM sale_date) AS yr,
       SUM(amount) AS total
FROM sales
GROUP BY CUBE(region, EXTRACT(YEAR FROM sale_date));
```

---

### Q210. GROUPING() function — distinguish subtotal rows
```sql
SELECT department, EXTRACT(YEAR FROM hire_date) AS yr,
       SUM(salary),
       GROUPING(department) AS is_dept_total,
       GROUPING(EXTRACT(YEAR FROM hire_date)) AS is_yr_total
FROM employees
GROUP BY ROLLUP(department, EXTRACT(YEAR FROM hire_date));
```

---

### Q211. JSONB — insert and query
```sql
-- Insert with JSONB
UPDATE products SET metadata = '{"color": "red", "weight": 1.5, "tags": ["sale","new"]}' WHERE id = 1;

-- Query JSONB field
SELECT name, metadata->>'color' AS color FROM products;

-- Query nested
SELECT name, (metadata->'specs'->>'ram') AS ram FROM products;
```

---

### Q212. JSONB — query with @> containment
```sql
-- Products with tag 'sale'
SELECT * FROM products
WHERE metadata @> '{"tags": ["sale"]}';
```

---

### Q213. JSONB — query with ? key existence
```sql
-- Products that have a 'color' key
SELECT * FROM products WHERE metadata ? 'color';

-- Has any of these keys
SELECT * FROM products WHERE metadata ?| ARRAY['color','weight'];

-- Has all of these keys
SELECT * FROM products WHERE metadata ?& ARRAY['color','weight'];
```

---

### Q214. JSONB — update nested field
```sql
UPDATE products
SET metadata = jsonb_set(metadata, '{color}', '"blue"', true)
WHERE id = 1;
```

---

### Q215. JSONB — remove a key
```sql
UPDATE products
SET metadata = metadata - 'color'
WHERE id = 1;
```

---

### Q216. JSONB — aggregate into object
```sql
SELECT jsonb_object_agg(name, salary) AS emp_salary_map
FROM employees
WHERE department = 'Engineering';
```

---

### Q217. JSONB — build array from rows
```sql
SELECT jsonb_agg(jsonb_build_object('name', name, 'salary', salary))
FROM employees
WHERE department = 'Engineering';
```

---

### Q218. JSONB — expand object to rows with jsonb_each
```sql
SELECT key, value
FROM products, jsonb_each(metadata)
WHERE id = 1;
```

---

### Q219. JSONB — filter inside array of objects
```sql
-- Products where any tag equals 'new'
SELECT * FROM products
WHERE EXISTS (
    SELECT 1 FROM jsonb_array_elements_text(metadata->'tags') AS t
    WHERE t = 'new'
);
```

---

### Q220. Full-Text Search — basic tsvector and tsquery
```sql
-- Search for products containing 'wireless' and 'keyboard'
SELECT name, description
FROM products
WHERE to_tsvector('english', description) @@ to_tsquery('english', 'wireless & keyboard');
```

---

### Q221. Full-Text Search — add GIN index
```sql
ALTER TABLE products ADD COLUMN search_vec TSVECTOR;

UPDATE products
SET search_vec = to_tsvector('english', COALESCE(name,'') || ' ' || COALESCE(description,''));

CREATE INDEX idx_products_fts ON products USING GIN(search_vec);

-- Query using the index
SELECT name FROM products WHERE search_vec @@ plainto_tsquery('english', 'wireless keyboard');
```

---

### Q222. Full-Text Search — ranking results
```sql
SELECT name,
  ts_rank(to_tsvector('english', description), query) AS rank
FROM products,
     to_tsquery('english', 'wireless & keyboard') AS query
WHERE to_tsvector('english', description) @@ query
ORDER BY rank DESC;
```

---

### Q223. PL/pgSQL — function with IF/ELSE and loop
```sql
CREATE OR REPLACE FUNCTION give_raise(dept TEXT, pct NUMERIC)
RETURNS INTEGER AS $$
DECLARE
    emp_count INTEGER := 0;
BEGIN
    IF pct <= 0 OR pct > 100 THEN
        RAISE EXCEPTION 'Percentage must be between 0 and 100';
    END IF;

    UPDATE employees
    SET salary = salary * (1 + pct / 100)
    WHERE department = dept
      AND is_active = TRUE;

    GET DIAGNOSTICS emp_count = ROW_COUNT;
    RETURN emp_count;
END;
$$ LANGUAGE plpgsql;

-- Usage:
SELECT give_raise('Engineering', 10);
```

---

### Q224. PL/pgSQL — loop over query results
```sql
CREATE OR REPLACE FUNCTION sync_dept_heads()
RETURNS VOID AS $$
DECLARE
    rec RECORD;
BEGIN
    FOR rec IN SELECT DISTINCT department FROM employees LOOP
        RAISE NOTICE 'Processing department: %', rec.department;
        -- do work per department
    END LOOP;
END;
$$ LANGUAGE plpgsql;
```

---

### Q225. TRIGGER — audit table on UPDATE
```sql
CREATE TABLE employees_audit (
    operation   CHAR(1),
    changed_at  TIMESTAMP DEFAULT NOW(),
    old_salary  NUMERIC,
    new_salary  NUMERIC,
    employee_id INT
);

CREATE OR REPLACE FUNCTION log_salary_change()
RETURNS TRIGGER AS $$
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
```sql
CREATE OR REPLACE FUNCTION prevent_salary_cut()
RETURNS TRIGGER AS $$
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

### Q227. ROW-LEVEL SECURITY (RLS) — employees see only their own row
```sql
ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

CREATE POLICY emp_self_view
ON employees
FOR SELECT
USING (id = current_setting('app.current_user_id')::INT);

-- Admin bypasses RLS:
ALTER TABLE employees FORCE ROW LEVEL SECURITY;
GRANT SELECT ON employees TO employee_role;
```

---

### Q228. Table partitioning — range partition by year
```sql
CREATE TABLE sales_partitioned (
    id        SERIAL,
    sale_date DATE NOT NULL,
    amount    NUMERIC(12,2),
    region    VARCHAR(100)
) PARTITION BY RANGE (sale_date);

CREATE TABLE sales_2023
    PARTITION OF sales_partitioned
    FOR VALUES FROM ('2023-01-01') TO ('2024-01-01');

CREATE TABLE sales_2024
    PARTITION OF sales_partitioned
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01');
```

---

### Q229. Table partitioning — list partition by region
```sql
CREATE TABLE sales_by_region (
    id     SERIAL,
    region VARCHAR(100) NOT NULL,
    amount NUMERIC(12,2)
) PARTITION BY LIST (region);

CREATE TABLE sales_north PARTITION OF sales_by_region FOR VALUES IN ('North', 'Northeast');
CREATE TABLE sales_south PARTITION OF sales_by_region FOR VALUES IN ('South', 'Southeast');
CREATE TABLE sales_default PARTITION OF sales_by_region DEFAULT;
```

---

### Q230. Table partitioning — hash partition
```sql
CREATE TABLE orders_hashed (
    id          SERIAL,
    customer_id INT,
    amount      NUMERIC(12,2)
) PARTITION BY HASH (customer_id);

CREATE TABLE orders_hash_0 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE orders_hash_1 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 1);
CREATE TABLE orders_hash_2 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 2);
CREATE TABLE orders_hash_3 PARTITION OF orders_hashed FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

---

### Q231. Lateral join with row-limiting per group
```sql
-- Top 3 products per category by price
SELECT p.category, t.name, t.price
FROM (SELECT DISTINCT category FROM products) p
CROSS JOIN LATERAL (
    SELECT name, price
    FROM products
    WHERE category = p.category
    ORDER BY price DESC
    LIMIT 3
) t;
```

---

### Q232. LATERAL with aggregation
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

### Q233. Window frame — RANGE vs ROWS
```sql
-- ROWS: physical rows
SELECT sale_date, amount,
  SUM(amount) OVER (ORDER BY sale_date ROWS BETWEEN 3 PRECEDING AND CURRENT ROW) AS rows_sum

-- RANGE: all rows with same date value
,SUM(amount) OVER (ORDER BY sale_date RANGE BETWEEN INTERVAL '3 days' PRECEDING AND CURRENT ROW) AS range_sum
FROM sales;
```

---

### Q234. Find median salary per department using PERCENTILE_CONT
```sql
SELECT department,
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY salary) AS median_salary
FROM employees
GROUP BY department;
```

---

### Q235. Detect salary anomalies — Z-score
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

### Q236. Unpivot using UNNEST and VALUES
```sql
-- Convert wide Q1/Q2/Q3/Q4 columns to tall format
SELECT region, quarter, sales
FROM (VALUES
  ('North', 'Q1', 100000),
  ('North', 'Q2', 150000),
  ('South', 'Q1', 90000),
  ('South', 'Q2', 110000)
) AS t(region, quarter, sales);
```

---

### Q237. Pivot using crosstab (tablefunc extension)
```sql
CREATE EXTENSION IF NOT EXISTS tablefunc;

SELECT * FROM crosstab(
    'SELECT department, EXTRACT(YEAR FROM hire_date)::TEXT, COUNT(*)
     FROM employees GROUP BY 1, 2 ORDER BY 1, 2',
    'SELECT DISTINCT EXTRACT(YEAR FROM hire_date)::TEXT FROM employees ORDER BY 1'
) AS ct(department TEXT, "2022" BIGINT, "2023" BIGINT, "2024" BIGINT);
```

---

### Q238. Transitive closure — all reachable nodes in a graph
```sql
-- Given: connections(from_id, to_id)
WITH RECURSIVE reachable(from_id, to_id) AS (
    SELECT from_id, to_id FROM connections
    UNION
    SELECT r.from_id, c.to_id
    FROM reachable r
    JOIN connections c ON r.to_id = c.from_id
)
SELECT DISTINCT from_id, to_id FROM reachable ORDER BY from_id, to_id;
```

---

### Q239. DISTINCT ON — get latest record per group
```sql
SELECT DISTINCT ON (customer_id)
    customer_id, order_date, amount, status
FROM orders
ORDER BY customer_id, order_date DESC;
```

---

### Q240. Window function — find records where value changes
```sql
SELECT name, department,
  LAG(department) OVER (PARTITION BY id ORDER BY hire_date) AS prev_dept
FROM employees
WHERE department IS DISTINCT FROM LAG(department) OVER (PARTITION BY id ORDER BY hire_date);
```

---

### Q241. Multi-level aggregation (aggregate of aggregates)
```sql
-- Average of department averages (weighted)
WITH dept_stats AS (
    SELECT department, AVG(salary) AS dept_avg, COUNT(*) AS cnt
    FROM employees GROUP BY department
)
SELECT ROUND(SUM(dept_avg * cnt) / SUM(cnt), 2) AS weighted_avg_salary
FROM dept_stats;
```

---

### Q242. Table inheritance
```sql
CREATE TABLE vehicles (
    id    SERIAL PRIMARY KEY,
    make  VARCHAR(100),
    model VARCHAR(100),
    year  INT
);

CREATE TABLE cars (wheel_count INT DEFAULT 4) INHERITS (vehicles);
CREATE TABLE trucks (payload_tons NUMERIC) INHERITS (vehicles);

-- Query all vehicles (includes cars and trucks)
SELECT * FROM vehicles;
-- Query only cars
SELECT * FROM ONLY cars;
```

---

### Q243. Advisory locks — application-level locking
```sql
-- Acquire a lock (returns TRUE if acquired)
SELECT pg_try_advisory_lock(42);

-- Release the lock
SELECT pg_advisory_unlock(42);

-- Transaction-level advisory lock (auto-released on commit/rollback)
SELECT pg_advisory_xact_lock(42);
```

---

### Q244. FOR UPDATE — pessimistic locking
```sql
BEGIN;
SELECT * FROM employees WHERE id = 1 FOR UPDATE;
-- Now no other transaction can update this row until COMMIT
UPDATE employees SET salary = 95000 WHERE id = 1;
COMMIT;
```

---

### Q245. FOR UPDATE SKIP LOCKED — queue processing
```sql
-- Multiple workers safely dequeue without waiting
SELECT * FROM orders
WHERE status = 'pending'
ORDER BY order_date
LIMIT 10
FOR UPDATE SKIP LOCKED;
```

---

### Q246. SAVEPOINT — partial rollback within a transaction
```sql
BEGIN;
INSERT INTO employees (name, salary) VALUES ('Vikram Singh', 60000);
SAVEPOINT after_vikram;

INSERT INTO employees (name, salary) VALUES ('Invalid', -1); -- will fail CHECK
ROLLBACK TO SAVEPOINT after_vikram;

-- Vikram is still inserted
COMMIT;
```

---

### Q247. EXPLAIN ANALYZE with BUFFERS
```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.name, d.name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.salary > 80000;
```

---

### Q248. Index-only scan — covering index
```sql
-- Index covers all columns needed — no heap access
CREATE INDEX idx_emp_dept_name ON employees(department) INCLUDE (name, salary);

EXPLAIN SELECT name, salary FROM employees WHERE department = 'Engineering';
-- Should show "Index Only Scan"
```

---

### Q249. GIN index on JSONB
```sql
CREATE INDEX idx_products_metadata ON products USING GIN(metadata);

-- Now this uses the index:
SELECT * FROM products WHERE metadata @> '{"color": "red"}';
```

---

### Q250. GIN index for full-text search
```sql
CREATE INDEX idx_products_fts ON products USING GIN(to_tsvector('english', description));

EXPLAIN SELECT * FROM products
WHERE to_tsvector('english', description) @@ plainto_tsquery('english', 'wireless');
```

---

### Q251. BRIN index — for naturally ordered data
```sql
-- Efficient for time-series tables where date increases naturally
CREATE INDEX idx_sales_date_brin ON sales USING BRIN(sale_date);
```

---

### Q252. pg_stat_statements — find slowest queries
```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;

SELECT query, calls, total_exec_time / 1000 AS total_sec,
       mean_exec_time / 1000 AS mean_sec, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;
```

---

### Q253. Detect table bloat
```sql
SELECT relname AS table,
  pg_size_pretty(pg_total_relation_size(oid)) AS total_size,
  n_dead_tup AS dead_tuples,
  n_live_tup AS live_tuples,
  ROUND(n_dead_tup * 100.0 / NULLIF(n_live_tup + n_dead_tup, 0), 2) AS dead_pct
FROM pg_stat_user_tables
ORDER BY dead_pct DESC NULLS LAST;
```

---

### Q254. VACUUM and ANALYZE
```sql
-- Reclaim dead tuple space
VACUUM employees;

-- Full vacuum (locks table, rewrites)
VACUUM FULL employees;

-- Update planner statistics
ANALYZE employees;

-- Both at once
VACUUM ANALYZE employees;
```

---

### Q255. Find unused indexes
```sql
SELECT schemaname, tablename, indexname, idx_scan
FROM pg_stat_user_indexes
WHERE idx_scan = 0
ORDER BY tablename, indexname;
```

---

### Q256. Find missing indexes — sequential scans on large tables
```sql
SELECT relname, seq_scan, seq_tup_read,
  idx_scan, seq_scan - idx_scan AS missing_index_est
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
  AND n_live_tup > 10000
ORDER BY seq_tup_read DESC;
```

---

### Q257. Connection pooling info — pg_stat_activity
```sql
SELECT state, COUNT(*) AS connections,
  usename, application_name
FROM pg_stat_activity
GROUP BY state, usename, application_name
ORDER BY connections DESC;
```

---

### Q258. Kill long-running queries
```sql
-- Find queries running more than 5 minutes
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE query != '<IDLE>'
  AND query_start < now() - INTERVAL '5 minutes';

-- Terminate gracefully
SELECT pg_terminate_backend(pid);

-- Force kill
SELECT pg_cancel_backend(pid);
```

---

### Q259. Detect deadlocks in pg_locks
```sql
SELECT blocked.pid AS blocked_pid,
       blocking.pid AS blocking_pid,
       blocked.query AS blocked_query,
       blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking
  ON blocking.pid = ANY(pg_blocking_pids(blocked.pid));
```

---

### Q260. Table statistics — correlation for index effectiveness
```sql
SELECT tablename, attname, correlation
FROM pg_stats
WHERE tablename = 'employees'
ORDER BY ABS(correlation) DESC;
```

---

### Q261. Dynamic SQL using EXECUTE in PL/pgSQL
```sql
CREATE OR REPLACE FUNCTION get_count_by(tbl TEXT, col TEXT, val TEXT)
RETURNS BIGINT AS $$
DECLARE
    result BIGINT;
BEGIN
    EXECUTE format('SELECT COUNT(*) FROM %I WHERE %I = $1', tbl, col)
    INTO result
    USING val;
    RETURN result;
END;
$$ LANGUAGE plpgsql;

SELECT get_count_by('employees', 'department', 'Engineering');
```

---

### Q262. Scheduled maintenance with pg_cron
```sql
CREATE EXTENSION IF NOT EXISTS pg_cron;

-- Refresh materialized view every hour
SELECT cron.schedule('refresh-stats', '0 * * * *',
  'REFRESH MATERIALIZED VIEW CONCURRENTLY dept_salary_stats');

-- Run VACUUM every night at 2am
SELECT cron.schedule('nightly-vacuum', '0 2 * * *', 'VACUUM ANALYZE employees');
```

---

### Q263. Custom aggregate function
```sql
CREATE OR REPLACE FUNCTION salary_product_state(state NUMERIC, sal NUMERIC)
RETURNS NUMERIC AS $$
    SELECT COALESCE(state, 1) * sal;
$$ LANGUAGE SQL;

CREATE AGGREGATE product_agg(NUMERIC) (
    SFUNC = salary_product_state,
    STYPE = NUMERIC,
    INITCOND = '1'
);

-- Usage:
SELECT department, product_agg(salary) FROM employees GROUP BY department;
```

---

### Q264. Write a function returning a TABLE
```sql
CREATE OR REPLACE FUNCTION dept_summary(dept_name TEXT)
RETURNS TABLE(emp_name TEXT, emp_salary NUMERIC, rank_in_dept BIGINT) AS $$
BEGIN
    RETURN QUERY
    SELECT name, salary,
           RANK() OVER (ORDER BY salary DESC)
    FROM employees
    WHERE department = dept_name;
END;
$$ LANGUAGE plpgsql;

SELECT * FROM dept_summary('Engineering');
```

---

### Q265. Event trigger — fire on DDL changes
```sql
CREATE OR REPLACE FUNCTION log_ddl_event()
RETURNS EVENT_TRIGGER AS $$
BEGIN
    RAISE NOTICE 'DDL event: % on %', tg_event, tg_tag;
END;
$$ LANGUAGE plpgsql;

CREATE EVENT TRIGGER ddl_logger
ON ddl_command_start
EXECUTE FUNCTION log_ddl_event();
```

---

### Q266. FDW — connect to another PostgreSQL database
```sql
CREATE EXTENSION postgres_fdw;

CREATE SERVER remote_db
FOREIGN DATA WRAPPER postgres_fdw
OPTIONS (host 'remote-host', port '5432', dbname 'analytics');

CREATE USER MAPPING FOR current_user
SERVER remote_db
OPTIONS (user 'remote_user', password 'secret');

IMPORT FOREIGN SCHEMA public
FROM SERVER remote_db INTO local_schema;

-- Now query remote table as if local:
SELECT * FROM local_schema.remote_employees;
```

---

### Q267. Logical replication — publish and subscribe
```sql
-- On publisher (source):
CREATE PUBLICATION emp_pub FOR TABLE employees, orders;

-- On subscriber (destination):
CREATE SUBSCRIPTION emp_sub
CONNECTION 'host=source-db port=5432 dbname=mydb user=replicator password=secret'
PUBLICATION emp_pub;
```

---

### Q268. COPY — fast bulk load
```sql
-- Load from CSV file
COPY employees(name, department, salary, hire_date)
FROM '/tmp/employees.csv'
WITH (FORMAT CSV, HEADER TRUE, DELIMITER ',');

-- Export to CSV
COPY (SELECT * FROM employees WHERE is_active = TRUE)
TO '/tmp/active_employees.csv'
WITH (FORMAT CSV, HEADER TRUE);
```

---

### Q269. Range type — prevent overlapping reservations
```sql
CREATE TABLE room_reservations (
    id       SERIAL PRIMARY KEY,
    room_id  INT,
    period   TSRANGE,
    EXCLUDE USING GIST (room_id WITH =, period WITH &&)
);

-- This insert will fail if same room is double-booked:
INSERT INTO room_reservations(room_id, period)
VALUES (101, '[2024-06-01 10:00, 2024-06-01 12:00)');
```

---

### Q270. Using RANGE operations
```sql
-- Check overlap
SELECT '[2024-01-01, 2024-06-30]'::DATERANGE && '[2024-04-01, 2024-12-31]'::DATERANGE;

-- Contains a date
SELECT '[2024-01-01, 2024-12-31]'::DATERANGE @> '2024-06-15'::DATE;

-- Union of two ranges
SELECT '[2024-01-01, 2024-06-30]'::DATERANGE + '[2024-05-01, 2024-12-31]'::DATERANGE;
```

---

### Q271. Generated columns
```sql
CREATE TABLE order_summary (
    id          SERIAL PRIMARY KEY,
    unit_price  NUMERIC(10,2),
    quantity    INT,
    total_price NUMERIC(12,2) GENERATED ALWAYS AS (unit_price * quantity) STORED
);

INSERT INTO order_summary(unit_price, quantity) VALUES (250.00, 3);
SELECT * FROM order_summary;
-- total_price = 750.00 (auto-computed)
```

---

### Q272. Custom domain types
```sql
CREATE DOMAIN email_address AS TEXT
CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE DOMAIN phone_number AS TEXT
CHECK (VALUE ~ '^\d{10}$');

CREATE DOMAIN positive_salary AS NUMERIC
CHECK (VALUE > 0);

-- Use in table:
CREATE TABLE staff (
    id     SERIAL PRIMARY KEY,
    email  email_address,
    phone  phone_number,
    salary positive_salary
);
```

---

### Q273. ENUM type
```sql
CREATE TYPE order_status AS ENUM ('pending', 'processing', 'shipped', 'delivered', 'cancelled');

ALTER TABLE orders ALTER COLUMN status TYPE order_status USING status::order_status;

-- Add a new value to enum
ALTER TYPE order_status ADD VALUE 'returned' AFTER 'delivered';
```

---

### Q274. Composite type
```sql
CREATE TYPE address AS (
    street  TEXT,
    city    TEXT,
    state   TEXT,
    pincode TEXT
);

CREATE TABLE customers_v2 (
    id      SERIAL PRIMARY KEY,
    name    TEXT,
    address address
);

INSERT INTO customers_v2(name, address)
VALUES ('Priya Nair', ROW('123 MG Road', 'Bangalore', 'Karnataka', '560001'));

SELECT name, (address).city, (address).pincode FROM customers_v2;
```

---

### Q275. Window function: ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
```sql
-- Each row sees the department total and its % share
SELECT name, department, salary,
  SUM(salary) OVER (PARTITION BY department
    ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING) AS dept_total,
  ROUND(salary * 100.0 /
    SUM(salary) OVER (PARTITION BY department
      ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING), 2) AS pct_share
FROM employees;
```

---

### Q276. Detect data skew in partitions
```sql
SELECT tableoid::REGCLASS AS partition, COUNT(*) AS row_count
FROM sales_partitioned
GROUP BY tableoid
ORDER BY row_count DESC;
```

---

### Q277. Find the busiest hours using generate_series
```sql
WITH hours AS (
    SELECT generate_series(0, 23) AS hr
),
order_counts AS (
    SELECT EXTRACT(HOUR FROM order_date) AS hr, COUNT(*) AS cnt
    FROM orders
    GROUP BY hr
)
SELECT h.hr, COALESCE(o.cnt, 0) AS orders
FROM hours h
LEFT JOIN order_counts o ON h.hr = o.hr
ORDER BY h.hr;
```

---

### Q278. Multi-column statistics for correlated columns
```sql
CREATE STATISTICS stats_emp_dept_sal ON department, salary FROM employees;
ANALYZE employees;

-- Now planner has better cardinality estimates for queries filtering both columns
EXPLAIN SELECT * FROM employees WHERE department = 'Engineering' AND salary > 70000;
```

---

### Q279. pg_notify / LISTEN — real-time notifications
```sql
-- In one session (listener):
LISTEN new_order;

-- In another session (notifier):
SELECT pg_notify('new_order', '{"order_id": 42, "amount": 1500}');

-- Or trigger-based notification:
CREATE OR REPLACE FUNCTION notify_new_order()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM pg_notify('new_order', row_to_json(NEW)::TEXT);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_new_order
AFTER INSERT ON orders
FOR EACH ROW EXECUTE FUNCTION notify_new_order();
```

---

### Q280. Outbox pattern — reliable event publishing
```sql
CREATE TABLE outbox (
    id         UUID DEFAULT gen_random_uuid() PRIMARY KEY,
    topic      TEXT NOT NULL,
    payload    JSONB NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    sent_at    TIMESTAMP
);

-- In the same transaction as the business write:
BEGIN;
INSERT INTO orders(customer_id, product_id, amount) VALUES (1, 5, 500);
INSERT INTO outbox(topic, payload)
VALUES ('order.created', jsonb_build_object('customer_id', 1, 'amount', 500));
COMMIT;
```

---

### Q281. Implement a sequence-based ID without SERIAL
```sql
CREATE SEQUENCE invoice_seq START 1000 INCREMENT 1 CACHE 10;

CREATE TABLE invoices (
    id         BIGINT DEFAULT nextval('invoice_seq') PRIMARY KEY,
    amount     NUMERIC(12,2),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Get next value without inserting
SELECT nextval('invoice_seq');

-- Reset sequence
ALTER SEQUENCE invoice_seq RESTART WITH 5000;
```

---

### Q282. Conditional INSERT with CTE (upsert pattern without ON CONFLICT)
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
SELECT id FROM inserted
UNION ALL
SELECT id FROM existing;
```

---

### Q283. CTE with RETURNING — chain insert → update
```sql
WITH new_order AS (
    INSERT INTO orders(customer_id, product_id, quantity, amount)
    VALUES (1, 5, 2, 1000)
    RETURNING id, product_id, quantity
)
UPDATE products
SET stock_qty = stock_qty - new_order.quantity
FROM new_order
WHERE products.id = new_order.product_id;
```

---

### Q284. Identify hot blocks — buffer cache hit ratio
```sql
SELECT relname,
  heap_blks_read,
  heap_blks_hit,
  ROUND(heap_blks_hit::NUMERIC / NULLIF(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS cache_hit_pct
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;
```

---

### Q285. Parallel query control
```sql
-- Enable parallel for a session
SET max_parallel_workers_per_gather = 4;

-- Force parallel plan
SET parallel_tuple_cost = 0;
SET parallel_setup_cost = 0;

-- Check if query uses parallel workers
EXPLAIN SELECT COUNT(*) FROM orders;
```

---

### Q286. Tablespace — move a table to different storage
```sql
CREATE TABLESPACE fast_ssd LOCATION '/mnt/ssd/pgdata';

-- Move table to the fast tablespace
ALTER TABLE orders SET TABLESPACE fast_ssd;

-- Move index
ALTER INDEX idx_orders_customer SET TABLESPACE fast_ssd;
```

---

### Q287. Temporal gap analysis — find inactive customers
```sql
WITH customer_activity AS (
    SELECT customer_id,
      order_date,
      LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order
    FROM orders
)
SELECT customer_id,
  order_date AS last_order_before_gap,
  next_order AS returned_on,
  next_order - order_date AS gap_days
FROM customer_activity
WHERE next_order - order_date > 90
ORDER BY gap_days DESC;
```

---

### Q288. Find the longest consecutive streak
```sql
WITH ranked AS (
    SELECT customer_id, sale_date,
      sale_date - ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY sale_date)::INT AS grp
    FROM (SELECT DISTINCT customer_id, order_date::DATE AS sale_date FROM orders) t
),
streaks AS (
    SELECT customer_id, grp, COUNT(*) AS streak_len
    FROM ranked
    GROUP BY customer_id, grp
)
SELECT customer_id, MAX(streak_len) AS longest_streak
FROM streaks
GROUP BY customer_id
ORDER BY longest_streak DESC;
```

---

### Q289. Recursive hierarchy — compute total reports count (size of subtree)
```sql
WITH RECURSIVE tree AS (
    SELECT id, manager_id, name, 0 AS depth
    FROM employees WHERE manager_id IS NULL
    UNION ALL
    SELECT e.id, e.manager_id, e.name, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
),
subtree_sizes AS (
    SELECT a.id, COUNT(b.id) AS total_reports
    FROM tree a
    JOIN tree b ON b.id != a.id
    WHERE EXISTS (
        WITH RECURSIVE sub AS (
            SELECT id FROM employees WHERE id = b.id
            UNION ALL
            SELECT e.id FROM employees e JOIN sub s ON e.manager_id = s.id
        )
        SELECT 1 FROM sub WHERE id = a.id LIMIT 1
    ) = FALSE
    GROUP BY a.id
)
SELECT e.name, COALESCE(s.total_reports, 0)
FROM employees e LEFT JOIN subtree_sizes s ON e.id = s.id;
```

---

### Q290. Schema evolution — zero-downtime column addition
```sql
-- Step 1: Add nullable column (fast, no rewrite)
ALTER TABLE orders ADD COLUMN delivery_at TIMESTAMP;

-- Step 2: Backfill in batches (non-blocking)
DO $$
DECLARE batch_size INT := 1000;
DECLARE last_id INT := 0;
BEGIN
    LOOP
        UPDATE orders
        SET delivery_at = order_date + INTERVAL '3 days'
        WHERE id > last_id AND delivery_at IS NULL
        RETURNING MAX(id) INTO last_id;
        EXIT WHEN NOT FOUND;
        PERFORM pg_sleep(0.01);
    END LOOP;
END $$;

-- Step 3: Set NOT NULL with a CHECK (validates without full scan in PG 11+)
ALTER TABLE orders ADD CONSTRAINT delivery_at_not_null CHECK (delivery_at IS NOT NULL) NOT VALID;
ALTER TABLE orders VALIDATE CONSTRAINT delivery_at_not_null;
ALTER TABLE orders ALTER COLUMN delivery_at SET NOT NULL;
ALTER TABLE orders DROP CONSTRAINT delivery_at_not_null;
```

---

### Q291. Custom operator
```sql
CREATE FUNCTION salary_greater(a NUMERIC, b NUMERIC) RETURNS BOOLEAN AS $$
    SELECT a > b * 1.5;
$$ LANGUAGE SQL;

CREATE OPERATOR >>> (LEFTARG = NUMERIC, RIGHTARG = NUMERIC, FUNCTION = salary_greater);

SELECT name, salary FROM employees WHERE salary >>> 50000;
-- Returns employees earning > 75000 (1.5x threshold)
```

---

### Q292. Partial aggregation and rollup with FILTER + CASE
```sql
SELECT
  department,
  COUNT(*)                                           AS total,
  COUNT(*) FILTER (WHERE salary > 80000)             AS high_earners,
  COUNT(*) FILTER (WHERE hire_date > '2023-01-01')   AS recent_hires,
  ROUND(AVG(salary), 2)                              AS avg_salary,
  ROUND(AVG(salary) FILTER (WHERE is_active), 2)     AS active_avg
FROM employees
GROUP BY ROLLUP(department);
```

---

### Q293. Graph query — shortest path using recursive CTE with cost
```sql
-- Given: routes(from_city, to_city, distance)
WITH RECURSIVE paths(from_city, to_city, distance, path, visited) AS (
    SELECT from_city, to_city, distance,
           ARRAY[from_city, to_city],
           ARRAY[from_city, to_city]
    FROM routes WHERE from_city = 'Mumbai'

    UNION ALL

    SELECT p.from_city, r.to_city, p.distance + r.distance,
           p.path || r.to_city,
           p.visited || r.to_city
    FROM paths p
    JOIN routes r ON p.to_city = r.from_city
    WHERE NOT r.to_city = ANY(p.visited)
)
SELECT from_city, to_city, distance, path
FROM paths WHERE to_city = 'Delhi'
ORDER BY distance LIMIT 1;
```

---

### Q294. Event sourcing query — rebuild current state from events
```sql
-- events table: (id, entity_id, event_type, payload JSONB, created_at)
WITH ordered_events AS (
    SELECT entity_id, event_type, payload,
           ROW_NUMBER() OVER (PARTITION BY entity_id ORDER BY created_at) AS seq
    FROM events
    WHERE entity_id = 42
),
aggregated AS (
    SELECT entity_id,
      MAX(CASE WHEN event_type = 'name_changed' THEN payload->>'name' END) AS name,
      SUM(CASE WHEN event_type = 'amount_added'
               THEN (payload->>'amount')::NUMERIC ELSE 0 END) AS balance,
      MAX(created_at) OVER (PARTITION BY entity_id) AS last_event_at
    FROM events
    WHERE entity_id = 42
    GROUP BY entity_id
)
SELECT * FROM aggregated;
```

---

### Q295. Query JSON array of objects — filter and extract
```sql
-- products.metadata = {"variants": [{"color":"red","qty":5}, {"color":"blue","qty":0}]}

-- Find products with at least one variant with qty > 0
SELECT p.name
FROM products p,
     jsonb_array_elements(p.metadata->'variants') AS v
WHERE (v->>'qty')::INT > 0
GROUP BY p.name;

-- List all in-stock variant colors per product
SELECT p.name, ARRAY_AGG(v->>'color') AS available_colors
FROM products p,
     jsonb_array_elements(p.metadata->'variants') AS v
WHERE (v->>'qty')::INT > 0
GROUP BY p.name;
```

---

### Q296. Streaming aggregation for time-series buckets
```sql
-- 5-minute buckets using date_trunc (or time_bucket with TimescaleDB)
SELECT DATE_TRUNC('minute', sale_date) -
       INTERVAL '1 minute' * (EXTRACT(MINUTE FROM sale_date)::INT % 5) AS bucket,
       COUNT(*) AS events,
       SUM(amount) AS total
FROM sales
GROUP BY bucket
ORDER BY bucket;
```

---

### Q297. Advanced RLS — multi-tenant isolation
```sql
CREATE TABLE tenant_data (
    id        SERIAL PRIMARY KEY,
    tenant_id INT NOT NULL,
    payload   JSONB
);

ALTER TABLE tenant_data ENABLE ROW LEVEL SECURITY;

CREATE POLICY tenant_isolation ON tenant_data
USING (tenant_id = current_setting('app.tenant_id')::INT);

-- Set tenant context at application level:
-- SET LOCAL app.tenant_id = '5';
```

---

### Q298. Parallel index build
```sql
-- PostgreSQL 11+ supports parallel CREATE INDEX
SET max_parallel_maintenance_workers = 4;

CREATE INDEX CONCURRENTLY idx_orders_amount_date
ON orders(amount DESC, order_date DESC);
-- CONCURRENTLY: builds without locking writes
```

---

### Q299. Table bloat recovery — pg_repack without locking
```sql
-- pg_repack is an extension (install separately)
-- It rewrites the table without holding an exclusive lock

-- Check if it's available
SELECT * FROM pg_available_extensions WHERE name = 'pg_repack';

-- Usage (run from psql or shell):
-- pg_repack -d mydb --table employees

-- Native alternative (acquires lock):
CLUSTER employees USING idx_employees_salary;
-- Or:
VACUUM FULL employees;
```

---

### Q300. Full production query: retention cohort analysis
```sql
WITH cohorts AS (
    -- Month each customer first ordered
    SELECT customer_id,
           DATE_TRUNC('month', MIN(order_date))::DATE AS cohort_month
    FROM orders
    GROUP BY customer_id
),
activity AS (
    -- Each customer's order months
    SELECT o.customer_id,
           c.cohort_month,
           DATE_TRUNC('month', o.order_date)::DATE AS order_month,
           (DATE_PART('year', o.order_date) - DATE_PART('year', c.cohort_month)) * 12
           + (DATE_PART('month', o.order_date) - DATE_PART('month', c.cohort_month))
           AS months_since_join
    FROM orders o
    JOIN cohorts c ON o.customer_id = c.customer_id
),
cohort_sizes AS (
    SELECT cohort_month, COUNT(DISTINCT customer_id) AS cohort_size
    FROM cohorts
    GROUP BY cohort_month
),
retention AS (
    SELECT a.cohort_month, a.months_since_join,
           COUNT(DISTINCT a.customer_id) AS retained
    FROM activity a
    GROUP BY a.cohort_month, a.months_since_join
)
SELECT r.cohort_month,
       cs.cohort_size,
       r.months_since_join,
       r.retained,
       ROUND(r.retained * 100.0 / cs.cohort_size, 1) AS retention_pct
FROM retention r
JOIN cohort_sizes cs ON r.cohort_month = cs.cohort_month
ORDER BY r.cohort_month, r.months_since_join;
```

---

## Summary

| Level  | Range    | Topics Covered |
|--------|----------|----------------|
| Easy   | Q1–Q100  | SELECT, WHERE, ORDER BY, GROUP BY, HAVING, JOINs, DDL, DML, string/date functions, CASE, COALESCE, subqueries, views |
| Medium | Q101–Q200 | Multi-table JOINs, UNION/INTERSECT/EXCEPT, CTEs, window functions (ROW_NUMBER, RANK, LAG, LEAD, SUM running total), EXISTS, pivots, UPSERT, LATERAL, string aggregation, EXPLAIN |
| Hard   | Q201–Q300 | Recursive CTEs, JSONB, full-text search, PL/pgSQL, triggers, RLS, partitioning, GROUPING SETS/ROLLUP/CUBE, gaps & islands, advisory locks, FDW, logical replication, event sourcing, cohort analysis, zero-downtime migrations |
