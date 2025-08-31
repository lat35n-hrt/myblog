+++
date = '2025-08-31T11:19:55+09:00'
draft = false
title = 'SQL Window関数の使い方 10選'
categories = ["sql"]
+++




|No|技術|概要|
|---|---|---|
|1|`HAVING COUNT = SUM(CASE)`|特定条件を満たすグループのみ抽出|
|2|`EXCEPT`|差分を取得|
|3|`DENSE_RANK()`|部署ごとのランキング|
|4|`LAG()`|直前との差分を取得|
|5|`MODE() WITHIN GROUP`|最頻値を取得|
|6|`ORDER BY + FETCH`|直近N件のデータ取得|
|7|`SUM(CASE...)`|ピボットテーブルの作成|
|8|`LEAD()`|連続欠番を検出|
|9|`ROW_NUMBER()`|各ユーザーの最新データを取得|
|10|`COALESCE() + LAG()`|NULL 値の補完|

エレガントな SQL の書き方は、**シンプルで可読性が高く、処理負荷を抑えることがポイント**です。これらのテクニックを活用すると、SQL の設計がより洗練され、メンテナンス性も向上します！





## 1. 条件を満たすグループのみ取得（HAVING）

**Goal:** `status` がすべて「完了」のプロジェクトを取得

**サンプルデータ（Tasks）**


| project_id | status |
| ---------- | ------ |
| 1          | 完了     |
| 1          | 完了     |
| 2          | 完了     |
| 2          | 未完了    |
| 3          | 完了     |


**SQL**

```sql
SELECT project_id
FROM Tasks
GROUP BY project_id
HAVING COUNT(*) = SUM(
	CASE WHEN status = '完了' THEN 1 ELSE 0 END
);
```
✅ PostgreSQL / MySQL / Oracle / SQL Server / SQLite
👉 全てのDBでそのまま動作可能


**結果**

| project_id |
| ---------- |
| 1          |
| 3          |


📌 **ポイント:** `COUNT(*)` と `SUM(CASE...)` を組み合わせて、特定条件を満たすグループのみを抽出。

---

## 2. 差分を求める（EXCEPT）

**Goal:** `Table_A` にあるが `Table_B` にないレコードを取得

**サンプルデータ**

- Table_A
    | id | name |
    |----|-------|
    | 1 | Alice |
    | 2 | Bob |
    | 3 | Carol |

- Table_B
    | id | name |
    |----|-------|
    | 2 | Bob |


**SQL**

```sql
SELECT id, name
FROM Table_A
EXCEPT
SELECT id, name
FROM Table_B;

```
- **PostgreSQL:** ✅ サポート
- **Oracle:** ✅ サポート
- **SQL Server:** ✅ サポート
- **SQLite:** ✅ サポート
- **MySQL:** ❌ 未サポート

**結果**

|id|name|
|---|---|
|1|Alice|
|3|Carol|

⚠️ 注意点
結果は自動で `DISTINCT` 扱い → 重複行が消える仕様を理解していないと混乱の元。

📌 **ポイント:** `EXCEPT` を使うことで、**集合の差分**を簡潔に取得可能。




👉 **MySQL 代替方法**

```sql
SELECT id, name
FROM Table_A
WHERE (id, name)
NOT IN (SELECT id, name FROM Table_B);
```


**⚠️ 注意点**

- `(id, name)` のようにタプル比較がサポートされないDB（古いMySQLなど）では動作しない。
- `NULL` が含まれると意図しない結果（全件除外）になる危険。


または

```sql
SELECT a.id, a.name
FROM Table_A a
LEFT JOIN Table_B b
  ON a.id = b.id
 AND a.name = b.name
WHERE b.id IS NULL;
```

**⚠️ 注意点**

- 少し冗長で、初学者には「なぜ b.id IS NULL？」と直感的でない。

- 読みやすさより「実務で確実に動く安全性」に寄った書き方。

まとめ

- **LEFT JOIN の直後** → A列 + B列（マッチしない場合はB列がNULL）

- **WHERE b.id IS NULL** → マッチしなかった行だけが残る（B列はNULLだらけ）

- **SELECT a.id, a.name** → B列を捨てて、純粋に「差集合 A−B」として出力



📌 総合レビュー（読みやすさ順）

1. **EXCEPT** → 一番シンプル。ただし DB依存。
2. **NOT IN** → 可読性良いが、NULLで破綻する可能性。
3. **LEFT JOIN + IS NULL** → 冗長だが安全でポータブル。



---

## 3. 部署ごとのランキング（DENSE_RANK）

**Goal:** 部署ごとに売上順位を付ける

**サンプルデータ（SalesData）**

|department|employee|sales|
|---|---|---|
|A|John|100|
|A|Alice|90|
|A|Bob|90|
|B|Carol|120|
|B|Dave|110|

**SQL**

```sql
SELECT department, employee, sales,
DENSE_RANK() OVER (PARTITION BY department ORDER BY sales DESC) AS rank
FROM SalesData;
```
- **PostgreSQL / Oracle / SQL Server:** ✅ 完全対応
- **MySQL:** ✅ 8.0 以降対応
- **SQLite:** ✅ 3.25 以降対応

**結果**

|department|employee|sales|rank|
|---|---|---|---|
|A|John|100|1|
|A|Alice|90|2|
|A|Bob|90|2|
|B|Carol|120|1|
|B|Dave|110|2|

📌 **ポイント:** `DENSE_RANK()` を使うと、
「同値は同順位、次の順位は飛ばさない」
順位付けが可能。

例: `RANK()` だと `1, 2, 2, 4` になるのに対して
`DENSE_RANK()` は `1, 2, 2, 3` になる。


🚀 `DENSE_RANK()` の代替 (ROW_NUMBER + 集計)

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

 📌 ポイント

- `COUNT(DISTINCT s2.sales)` を使って「その部署の中で、自分より大きい or 同じ売上の数」を数える。

- これにより `DENSE_RANK()` と同じ順位が得られる。

- **ROW_NUMBER()** だけだと「同点も別順位」になってしまうが、
    サブクエリで **順位 = 値の distinct count** に変換することで対応可能。

---

## 4. 直前との差分を求める（LAG）

**Goal:** 商品ごとに前回の価格差を計算

**サンプルデータ（Sales）**

|product_id|sale_date|price|
|---|---|---|
|P1|2024-01-01|100|
|P1|2024-02-01|120|
|P1|2024-03-01|115|

**SQL**

``` sql
SELECT product_id, sale_date, price,
price - LAG(price) OVER (PARTITION BY product_id ORDER BY sale_date) AS price_change
FROM Sales;
```
- **PostgreSQL / Oracle / SQL Server:** ✅ 完全対応
- **MySQL:** ✅ 8.0 以降対応
- **SQLite:** ✅ 3.25 以降対応


**結果**

| product_id | sale_date  | price | price_change |
| ---------- | ---------- | ----- | ------------ |
| P1         | 2024-01-01 | 100   | NULL         |
| P1         | 2024-02-01 | 120   | 20           |
| P1         | 2024-03-01 | 115   | -5           |

📌 **ポイント:**
- `LAG()` で前回データを取得し、**差分計算**を実現。
- _最初の行は前回データが存在しないため、`LAG()` の戻り値が `NULL` になる」_。




---

## 5. 最頻値を求める（MODE）

**Goal:** カテゴリごとに最も売れた商品を取得

**サンプルデータ（Sales）**

|category|product_id|
|---|---|
|A|P1|
|A|P2|
|A|P2|
|B|P3|
|B|P3|
|B|P4|

**SQL**
```sql
SELECT category,
MODE() WITHIN GROUP (ORDER BY product_id) AS most_sold_product
FROM Sales
GROUP BY category;
```
- **Oracle:** ✅ サポート
- **PostgreSQL:** ⚠️ `MODE()` は標準では未サポート。ただし `percentile_disc()` などで代替可能
- **MySQL / SQL Server / SQLite:** ❌ 未サポート

**結果**

|category|most_sold_product|
|---|---|
|A|P2|
|B|P3|


👉 **代替方法（全DBで使える最頻値取得）**

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
_「最頻値が複数ある場合、`RANK()` を使うとすべて返却される。1件だけに絞るなら `ROW_NUMBER()` を使う」_。



📌 **ポイント:**

- `MODE() WITHIN GROUP` はシンプルだが Oracle 限定。
- 代替SQLは **全DBで動作可能**。
	- 複数の最頻値がある場合、`RANK()` によりすべて返却される。
	    - 1つに絞りたい場合は `ROW_NUMBER()` に変更。




---

## 6. 直近の N 件（FETCH/TOP/LIMIT）

**Goal:** 直近10件の注文を取得

**サンプルデータ（Orders）**

| order_id | order_date |     |
| -------- | ---------- | --- |
| 101      | 2024-01-01 |     |
| 102      | 2024-01-05 |     |
| …        | …          |     |
| 120      | 2024-02-01 |     |

**SQL**

👉標準SQL（PostgreSQL / Oracle）
```sql
SELECT *
FROM Orders
ORDER BY order_date DESC
FETCH FIRST 10 ROWS ONLY;
```


👉 **SQL Server**
```sql
-- 従来構文
SELECT TOP 10 *
FROM Orders
ORDER BY order_date DESC;

-- 標準に近い構文（SQL Server 2012+）
SELECT *
FROM Orders
ORDER BY order_date DESC
OFFSET 0 ROWS FETCH NEXT 10 ROWS ONLY;
```

👉 **MySQL / SQLite**
```sql
SELECT *
FROM Orders
ORDER BY order_date
DESC LIMIT 10;
```


**結果**
（最新日付から 10 件）


📌 **ポイント:**

- `FETCH FIRST` は標準SQL準拠だが、各DBでは `LIMIT` や `TOP` がよく使われる。

- **必ず ORDER BY を併用すること**（でないと「直近」が保証されない）。


---

## 7. ピボット（CASE）

**Goal:** 部署ごとに `status` の件数を横持ちにする

**サンプルデータ（Tasks）**

|department|status|
|---|---|
|A|完了|
|A|未完了|
|B|完了|

SQL（全DB共通）
```sql
SELECT department,
       SUM(CASE WHEN status = '完了' THEN 1 ELSE 0 END) AS completed,
       SUM(CASE WHEN status = '未完了' THEN 1 ELSE 0 END) AS pending
FROM Tasks
GROUP BY department;
```


別解（全DB共通：COUNTを活用）
``` SQL
SELECT department,
       COUNT(CASE WHEN status = '完了' THEN 1 END) AS completed,
       COUNT(CASE WHEN status = '未完了' THEN 1 END) AS pending
FROM Tasks
GROUP BY department;
```


✅ PostgreSQL / MySQL / Oracle / SQL Server / SQLite
👉 全てのDBでそのまま動作可能


**結果**

|department|completed|pending|
|---|---|---|
|A|1|1|
|B|1|0|


📌 **ポイント:**

- `CASE` による条件集計は全DBで利用可能 → クロスDB最強のテクニック。

- **固定カテゴリが少ない場合:** 今回のようにシンプルな `CASE` が最適。

- **カテゴリが多い場合:** Oracle / SQL Server では `PIVOT` 句を使うとコードが短くなる。


---

## 8. 連続欠番を特定（LEAD）

**Goal:** ID の飛びを検出

**サンプルデータ（Orders）**

|id|
|---|
|1|
|2|
|4|

**SQL**

```sql
SELECT *
FROM (
  SELECT id, LEAD(id) OVER (ORDER BY id) AS next_id
  FROM Orders
) t
WHERE next_id IS NOT NULL
  AND next_id <> id + 1;
```
- **PostgreSQL / Oracle ✅ 対応
- **SQL Server 2012:** ✅ 以降対応
- **MySQL:** ✅ 8.0 以降対応
- **SQLite:** ✅ 3.25 以降対応


**結果**

|id|next_id|
|---|---|
|2|4|


📌 **ポイント:**

- `LEAD()` を活用することで、**次の行との連番の欠落**を簡単に特定できる。

- 簡単に「欠番がある行」と「次のID」を取得可能。



**⚠️ 注意点**

`WHERE` 句で直接 LEAD() を使用できません。
```sql
--  使用できない例
SELECT id,
	LEAD(id) OVER (ORDER BY id) AS next_id
FROM Orders
WHERE LEAD(id) OVER (ORDER BY id) <> id + 1;
```

- PostgreSQL / Oracle / SQL Server/SQLite:  ❌ Where句で LEAD() 使用できない





---

## 9. 最新注文を取得（ROW_NUMBER）

**Goal:** 顧客ごとに最新注文だけ取得

**サンプルデータ（Orders）**

|order_id|customer_id|order_date|
|---|---|---|
|1|C1|2024-01-01|
|2|C1|2024-02-01|
|3|C2|2024-01-10|

**SQL**

```sql
SELECT order_id, customer_id, order_date
FROM (
	SELECT order_id, customer_id, order_date,
		ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_date DESC) AS row_num
	FROM Orders ) AS subquery
WHERE row_num = 1;
```
- **PostgreSQL / Oracle / SQL Server:** ✅ 完全対応
- **MySQL:** ✅ 8.0 以降対応
- **SQLite:** ✅ 3.25 以降対応


**結果**

|order_id|customer_id|order_date|
|---|---|---|
|2|C1|2024-02-01|
|3|C2|2024-01-10|


📌 **ポイント:** `ROW_NUMBER()` を使い、**各 `customer_id` の最新の注文を 1 件だけ取得**。


---

## 10. NULL 値の補完（COALESCE + LAG）

**Goal:** 価格が NULL の場合、直近の値を補完

**サンプルデータ（Sales）**

|product_id|sale_date|price|
|---|---|---|
|P1|2024-01-01|100|
|P1|2024-02-01|NULL|
|P1|2024-03-01|120|

**SQL**
```sql
SELECT product_id, sale_date,
	COALESCE(price, LAG(price) OVER (PARTITION BY product_id ORDER BY sale_date)) AS filled_price
FROM Sales;
```
- **PostgreSQL / Oracle / SQL Server:** ✅ 完全対応
- **MySQL:** ✅ 8.0 以降対応
- **SQLite:** ✅ 3.25 以降対応


**結果**

|product_id|sale_date|filled_price|
|---|---|---|
|P1|2024-01-01|100|
|P1|2024-02-01|100|
|P1|2024-03-01|120|


このクエリでは **日付順に並べた各 product_id ごとに**、

- `price` が **NULL でなければその値を使う**
- `price` が **NULL なら 1つ前の行 (LAG) の値を代わりに使う**

もし連続で NULL が続くと、1回目は補完されても2回目はまた NULL になります。


📌 **ポイント:** `COALESCE()` と `LAG()` を組み合わせて、**NULL 値の補完**を実現。



💡 **補足テクニック**

- 連続 NULL を完全に前値で埋めたい場合（累積補完）は、再帰CTEを使った次のようなアプローチもあります：

``` sql
WITH RECURSIVE filled AS (
  SELECT product_id, sale_date, price AS filled_price
  FROM Sales
  WHERE sale_date = (SELECT MIN(sale_date) FROM Sales s2 WHERE s2.product_id = Sales.product_id)
  UNION ALL
  SELECT s.product_id, s.sale_date,
         COALESCE(s.price, f.filled_price)
  FROM Sales s
  JOIN filled f
    ON s.product_id = f.product_id
   AND s.sale_date = f.sale_date + interval '1 month'
)
SELECT * FROM filled;
```

この方法なら、連続 NULL もすべて前の値で埋められます。


---



