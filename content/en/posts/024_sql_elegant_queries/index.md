+++
date = '2025-08-31T11:20:04+09:00'
draft = false
title = 'SQL Window Functions: 10 Practical Examples'
categories = ["sql"]
+++


| No | Technique | Summary |
|----|-----------|---------|
| 1  | `HAVING COUNT = SUM(CASE)` | Extract only groups that meet specific conditions |
| 2  | `EXCEPT` | Retrieve differences between tables |
| 3  | `DENSE_RANK()` | Ranking within each department |
| 4  | `LAG()` | Difference from the previous row |
| 5  | `MODE() WITHIN GROUP` | Find the most frequent value |
| 6  | `ORDER BY + FETCH` | Get the latest N rows |
| 7  | `SUM(CASE...)` | Create pivot tables |
| 8  | `LEAD()` | Detect missing consecutive values |
| 9  | `ROW_NUMBER()` | Get the latest record per user |
| 10 | `COALESCE() + LAG()` | Fill in NULL values |

Writing elegant SQL means **keeping queries simple, readable, and efficient**.
By applying these techniques, you can make your SQL designs more refined and easier to maintain.


---

## 1. Extract groups that meet conditions (HAVING)

**Goal:** Get projects where all tasks have `status = 'Complete'`

**Sample Data (Tasks)**

| project_id | status   |
|------------|----------|
| 1          | Complete |
| 1          | Complete |
| 2          | Complete |
| 2          | Incomplete |
| 3          | Complete |

**SQL**
```sql
SELECT project_id
FROM Tasks
GROUP BY project_id
HAVING COUNT(*) = SUM(
    CASE WHEN status = 'Complete' THEN 1 ELSE 0 END
);
```

✅ PostgreSQL / MySQL / Oracle / SQL Server / SQLite
👉 Works as-is across all databases

Result

| project\_id |
| ----------- |
| 1           |
| 3           |


📌 Point: Combine COUNT(*) and SUM(CASE...) to filter only groups meeting specific criteria.

## 2. Retrieve differences (EXCEPT)

Goal: Get rows that exist in Table_A but not in Table_B

Sample Data

Table_A

| id | name  |
|----|-------|
| 1  | Alice |
| 2  | Bob   |
| 3  | Carol |


Table_B

| id | name |
|----|------|
| 2  | Bob  |


SQL

```sql
SELECT id, name
FROM Table_A
EXCEPT
SELECT id, name
FROM Table_B;
```


PostgreSQL / Oracle / SQL Server / SQLite: ✅ Supported

MySQL: ❌ Not supported

Result

| id | name  |
| -- | ----- |
| 1  | Alice |
| 3  | Carol |


⚠️ Note: EXCEPT implicitly applies DISTINCT. Duplicate rows are removed.

📌 Point: EXCEPT provides a concise way to compute set differences.

👉 MySQL Alternative

```sql
SELECT id, name
FROM Table_A
WHERE (id, name)
NOT IN (SELECT id, name FROM Table_B);
```


⚠️ Caveats:

Tuple comparison (id, name) may not work in older MySQL versions.

If NULL is present, it can unexpectedly exclude all rows.

👉 Another alternative (portable)


```sql
SELECT a.id, a.name
FROM Table_A a
LEFT JOIN Table_B b
  ON a.id = b.id
 AND a.name = b.name
WHERE b.id IS NULL;
```

⚠️ Caveats:

Slightly verbose and less intuitive (b.id IS NULL may confuse beginners).

However, safer and more portable across DBs.

📌 Comparison (readability vs. safety):

EXCEPT → simplest but DB-dependent

NOT IN → readable but fragile with NULL

LEFT JOIN + IS NULL → verbose but safest and most portable

## 3. Ranking by department (DENSE_RANK)

Goal: Rank employees by sales within each department

Sample Data (SalesData)

| department | employee | sales |
| ---------- | -------- | ----- |
| A          | John     | 100   |
| A          | Alice    | 90    |
| A          | Bob      | 90    |
| B          | Carol    | 120   |
| B          | Dave     | 110   |


SQL

```sql
SELECT department, employee, sales,
DENSE_RANK() OVER (PARTITION BY department ORDER BY sales DESC) AS rank
FROM SalesData;
```

PostgreSQL / Oracle / SQL Server: ✅ Supported

MySQL: ✅ Supported (8.0+)

SQLite: ✅ Supported (3.25+)

Result

| department | employee | sales | rank |
| ---------- | -------- | ----- | ---- |
| A          | John     | 100   | 1    |
| A          | Alice    | 90    | 2    |
| A          | Bob      | 90    | 2    |
| B          | Carol    | 120   | 1    |
| B          | Dave     | 110   | 2    |


📌 Point:

DENSE_RANK() assigns the same rank to ties and does not skip the next rank.

Compare with RANK():

RANK() → 1, 2, 2, 4

DENSE_RANK() → 1, 2, 2, 3

🚀 Manual alternative using subquery

```sql
SELECT s.department,
       s.employee,
       s.sales,
       (
         SELECT COUNT(DISTINCT s2.sales)
         FROM SalesData s2
         WHERE s2.department = s.department
           AND s2.sales >= s.sales
       ) AS rank
FROM SalesData s
ORDER BY s.department, rank;
```

📌 Uses COUNT(DISTINCT s2.sales) to mimic DENSE_RANK.

## 4. Difference from previous row (LAG)

Goal: Calculate the price change compared to the previous sale per product

Sample Data (Sales)
| product\_id | sale\_date | price |
| ----------- | ---------- | ----- |
| P1          | 2024-01-01 | 100   |
| P1          | 2024-02-01 | 120   |
| P1          | 2024-03-01 | 115   |


SQL

```sql
SELECT product_id, sale_date, price,
price - LAG(price) OVER (PARTITION BY product_id ORDER BY sale_date) AS price_change
FROM Sales;
```
PostgreSQL / Oracle / SQL Server: ✅ Supported

MySQL: ✅ Supported (8.0+)

SQLite: ✅ Supported (3.25+)

Result
| product\_id | sale\_date | price | price\_change |
| ----------- | ---------- | ----- | ------------- |
| P1          | 2024-01-01 | 100   | NULL          |
| P1          | 2024-02-01 | 120   | 20            |
| P1          | 2024-03-01 | 115   | -5            |


📌 Point:

LAG() retrieves the previous row’s value.

The first row has no previous row, so it returns NULL.


## 5. Find the Mode (Most Frequent Value)

Goal: Get the most sold product per category

Sample Data (Sales)

| category | product\_id |
| -------- | ----------- |
| A        | P1          |
| A        | P2          |
| A        | P2          |
| B        | P3          |
| B        | P3          |
| B        | P4          |


SQL
```sql
SELECT category,
MODE() WITHIN GROUP (ORDER BY product_id) AS most_sold_product
FROM Sales
GROUP BY category;
```

Oracle: ✅ Supported

PostgreSQL: ⚠️ MODE() not supported by default (can be emulated with percentile_disc())

MySQL / SQL Server / SQLite: ❌ Not supported

Result
| category | most\_sold\_product |
| -------- | ------------------- |
| A        | P2                  |
| B        | P3                  |

👉 Cross-DB Alternative (works everywhere)
```sql
SELECT category, product_id
FROM (
  SELECT category, product_id,
         RANK() OVER (PARTITION BY category ORDER BY COUNT(*) DESC) AS rnk
  FROM Sales
  GROUP BY category, product_id
) t
WHERE rnk = 1;
```

📌 Point:

MODE() WITHIN GROUP is elegant but Oracle-only.

The RANK() approach works across all databases and returns all top candidates when ties exist.

Use ROW_NUMBER() if you only want one row.

## 6. Fetch the Most Recent N Rows (FETCH / TOP / LIMIT)

Goal: Get the latest 10 orders

Sample Data (Orders)

| order\_id | order\_date |
| --------- | ----------- |
| 101       | 2024-01-01  |
| 102       | 2024-01-05  |
| ...       | ...         |
| 120       | 2024-02-01  |


SQL

👉 Standard SQL (PostgreSQL / Oracle)
```sql
SELECT *
FROM Orders
ORDER BY order_date DESC
FETCH FIRST 10 ROWS ONLY;
```

👉 SQL Server
```sql
-- Legacy style
SELECT TOP 10 *
FROM Orders
ORDER BY order_date DESC;

-- Standard-like (SQL Server 2012+)
SELECT *
FROM Orders
ORDER BY order_date DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

👉 MySQL / SQLite
```sql
SELECT *
FROM Orders
ORDER BY order_date DESC
LIMIT 10;
```

📌 Point:

Always combine with ORDER BY, otherwise “latest” is not guaranteed.

Syntax differs by DBMS, but the concept is universal.

## 7. Pivot with CASE

Goal: Convert status into columns per department

Sample Data (Tasks)


| department | status    |
| ---------- | --------- |
| A          | Completed |
| A          | Pending   |
| B          | Completed |


SQL (Cross-DB Compatible)

```sql
SELECT department,
       SUM(CASE WHEN status = 'Completed' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN status = 'Pending' THEN 1 ELSE 0 END) AS pending
FROM Tasks
GROUP BY department;
```

Alternative (COUNT-based, also portable):
```sql
SELECT department,
       COUNT(CASE WHEN status = 'Completed' THEN 1 END) AS completed,
       COUNT(CASE WHEN status = 'Pending' THEN 1 END) AS pending
FROM Tasks
GROUP BY department;
```


Result
| department | completed | pending |
| ---------- | --------- | ------- |
| A          | 1         | 1       |
| B          | 1         | 0       |


📌 Point:

Conditional aggregation via CASE is the most portable way to pivot data.

For many categories, Oracle / SQL Server’s PIVOT clause can reduce verbosity.

## 8. Detect Gaps in Sequences (LEAD)

Goal: Identify missing IDs

Sample Data (Orders)
| id |
| -- |
| 1  |
| 2  |
| 4  |


SQL

```sql
SELECT *
FROM (
  SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id
  FROM Orders
) t
WHERE next_id IS NOT NULL
  AND next_id <> id + 1;
```

PostgreSQL / Oracle: ✅ Supported

SQL Server 2012+: ✅ Supported

MySQL 8.0+: ✅ Supported

SQLite 3.25+: ✅ Supported

Result
| id | next\_id |
| -- | -------- |
| 2  | 4        |


📌 Point:

LEAD() makes it easy to spot sequence gaps by comparing the current row with the next row.

Cannot be used directly in WHERE—wrap it in a subquery or CTE.

## 9. Get the Latest Row per Group (ROW_NUMBER)

Goal: Get the latest order per customer

Sample Data (Orders)

| order\_id | customer\_id | order\_date |
| --------- | ------------ | ----------- |
| 1         | C1           | 2024-01-01  |
| 2         | C1           | 2024-02-01  |
| 3         | C2           | 2024-01-10  |


SQL

```sql
SELECT order_id, customer_id, order_date
FROM (
  SELECT order_id, customer_id, order_date,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS row_num
  FROM Orders
) t
WHERE row_num = 1;
```


Result

| order\_id | customer\_id | order\_date |
| --------- | ------------ | ----------- |
| 2         | C1           | 2024-02-01  |
| 3         | C2           | 2024-01-10  |


📌 Point:

ROW_NUMBER() ensures exactly one row per group, sorted by your chosen column.

Ideal for deduplication and “latest record” queries.

## 10. Fill NULLs with Previous Value (COALESCE + LAG)

Goal: Fill missing prices with the last known value

Sample Data (Sales)

| product\_id | sale\_date | price |
| ----------- | ---------- | ----- |
| P1          | 2024-01-01 | 100   |
| P1          | 2024-02-01 | NULL  |
| P1          | 2024-03-01 | 120   |


SQL
```sql
SELECT product_id, sale_date,
       COALESCE(price, LAG(price) OVER (PARTITION BY product_id ORDER BY sale_date)) AS filled_price
FROM Sales;
```

Result

| product\_id | sale\_date | filled\_price |
| ----------- | ---------- | ------------- |
| P1          | 2024-01-01 | 100           |
| P1          | 2024-02-01 | 100           |
| P1          | 2024-03-01 | 120           |


📌 Point:

COALESCE() chooses the first non-NULL value.

Together with LAG(), you can replace missing values with the previous one.

For continuous filling (multiple NULLs), consider recursive CTEs.