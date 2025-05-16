+++
date = '2025-05-16T16:40:36+09:00'
draft = false
title = 'PHPxSQL Dynamic Query'
categories = ["sql"]
+++


[PHP × SQL] The Classic Technique for Dynamically Adding WHERE Conditions: How and Why to Use 1=1
When building web applications, you often encounter situations where you need to dynamically construct SQL queries based on user-specified conditions. This is especially common in search screens and filter features, where multiple optional inputs are available.

In such cases, a widely-used trick is to begin your query with a seemingly magical condition: WHERE 1=1. In this article, we'll explore why this works and how to use it effectively, with a practical example.

■ Why Use 1=1?
When dynamically constructing SQL queries in PHP, you'll often append conditions to a base query string using string concatenation.

However, when multiple conditions are added, you run into the issue of whether to prepend AND or not. For example:

```php
$sql = 'SELECT * FROM books WHERE';
if ($minPrice !== null) {
    $sql .= ' price >= :min_price';
}
if ($maxPrice !== null) {
    $sql .= ' AND price <= :max_price';
}

```


In this example, you need to consider whether to add AND based on the number and order of conditions, which quickly becomes cumbersome and error-prone.

That’s where the 1=1 trick comes in.

```sql
WHERE 1=1

```

This is a dummy condition that is always TRUE, and it acts as a placeholder for the first condition. Once it's in place, you can safely add AND before every real condition without worrying about logic errors.

■ Practical Example: Filtering Books by Price Range and Publish Year
Here’s a sample PHP snippet that builds a SQL query based on three optional filter parameters: $minPrice, $maxPrice, and $year.



```php

// Build the SQL query dynamically
$sql = 'SELECT * , is_borrowed FROM books WHERE 1=1';

$params = [];

if ($minPrice !== null) {
    $sql .= ' AND price >= :min_price';
    $params[':min_price'] = $minPrice;
}

if ($maxPrice !== null) {
    $sql .= ' AND price <= :max_price';
    $params[':max_price'] = $maxPrice;
}

if ($year !== null) {
    $sql .= ' AND YEAR(publish) = :year'; // MySQL specific: YEAR() function
    $params[':year'] = $year;
}

```

If, for example, only $minPrice and $year are specified, the resulting SQL will look like this:


```sql
SELECT * , is_borrowed FROM books
WHERE 1=1
  AND price >= :min_price
  AND YEAR(publish) = :year

```

Thanks to the 1=1 base condition, you can consistently prepend AND to all your conditional clauses, keeping the logic and code clean.

■ Summary: 1=1 Balances Readability and Flexibility
Using WHERE 1=1 is a simple yet powerful technique for dynamically constructing SQL queries in a safe and readable way. It's commonly used in production code.

Benefits of this approach include:
You don’t have to worry about when or where to insert AND

The logic remains stable even if the number or order of conditions changes

The resulting SQL is easier to read and maintain

At first glance, 1=1 may seem like a meaningless condition—but for developers working with dynamic SQL, it’s a handy and reliable placeholder that makes your code more maintainable.

