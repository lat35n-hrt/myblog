+++
date = '2025-05-14T19:31:32+09:00'
draft = false
title = 'PHP Secure Delete'
categories = ["PHP"]
+++


### ✅ Key Security Considerations When Implementing Delete Functionality in PHP
In the previous article, we discussed how to implement CSRF tokens and authentication checks. As a practical follow-up, this article summarizes the key security measures you should take when implementing a custom delete operation in PHP.

<br>

### 1. 🔐 CSRF Token Validation

``` php
if (empty($_POST['token']) || !hash_equals($_SESSION['token'], $_POST['token'])) {
    echo "Invalid CSRF token.";
    exit;
}

```

This snippet ensures that requests originate from valid sessions, not from malicious third-party sites. The token in the session is matched against the token submitted via POST — a critical step in defending against Cross-Site Request Forgery (CSRF) attacks.

<br>

### 2. 🚫 Use POST Instead of GET

``` php
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    echo "Invalid request method.";
    exit;
}

```

Using GET for delete operations is dangerous for several reasons:
The resource can be deleted simply by visiting a URL
Browser previews or link sharing can unintentionally trigger the deletion
Therefore, we explicitly enforce POST to prevent accidental or unauthorized deletions.

<br>

### 3. ✅ User Permission Check

``` php
if ($_SESSION['role'] !== 'admin' && $_SESSION['role'] !== 'editor') {
    echo "You do not have permission to delete books.";
    exit;
}

```

To prevent unauthorized or accidental deletions, only users with the admin or editor role are allowed to perform delete operations.

<br>

### 4. 🔒 SQL Injection Protection

``` php
$sql = 'DELETE FROM books WHERE id = :id';
$stmt = $dbh->prepare($sql);
$stmt->bindValue(':id', $id, PDO::PARAM_INT);
$stmt->execute();
```

We use prepared statements with bindValue() to ensure user input is treated safely, effectively preventing SQL injection attacks.

<br>

### 5. 📦 Record Existence Check and Transaction Handling


``` php
$dbh->beginTransaction();

// Check if the record exists
// ...

// Perform delete
// ...

$dbh->commit();

```

Before deletion, verify that the target record exists. By wrapping the process in a transaction, you ensure consistency — if an error occurs, rollback() can revert the operation safely.

<br>

### 6. 🧼 XSS Protection with str2html()

``` php
echo "An error occurred: " . str2html($e->getMessage());

```


Use the custom str2html() function to sanitize any output displayed to users. This prevents HTML or JavaScript from being injected and executed.

``` php
function str2html(string $arg_input): string {
    return htmlspecialchars($arg_input, ENT_QUOTES, 'UTF-8');
}

```

<br>

### 📝 Summary
Delete operations may seem straightforward at first, but poor implementation can lead to serious vulnerabilities or accidental data loss. In this implementation, we’ve taken care to include the following security best practices:

- Enforcing the use of the POST method

- CSRF token validation

- Permission checks for authorized users

- SQL injection prevention via prepared statements

- Ensuring data consistency with transactions

- XSS protection for output sanitization

By following these guidelines, you can ensure your delete functionality is both robust and secure.

<br>

### Sample codes

delete.php
```php
<?php
session_start();
require_once __DIR__ . '/../shared/login_check.php'; // Login checker
require_once __DIR__ . '/../shared/functions.php'; // Contains functions like str2html() and db_open()

// Check if the request method is POST
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    echo "Invalid request method.";
    exit;
}

// Validate CSRF token
if (empty($_POST['token']) || empty($_SESSION['token']) || !hash_equals($_SESSION['token'], $_POST['token'])) {
    echo "Invalid CSRF token.";
    exit;
}

// Validate and sanitize the 'id'
$id = filter_input(INPUT_POST, 'id', FILTER_VALIDATE_INT);
if ($id === false) {
    echo "Invalid ID.";
    exit;
}

// Check user role for permission
if ($_SESSION['role'] !== 'admin' && $_SESSION['role'] !== 'editor') {
    echo "You do not have permission to delete books.";
    exit;
}


try {
    // Database connection
    $dbh = db_open();

    // Start transaction: Auto-Commit is now disabled
    $dbh->beginTransaction();

    // Check if the record exists
    $sql = 'SELECT COUNT(*) FROM books WHERE id = :id';
    $stmt = $dbh->prepare($sql);
    $stmt->bindParam(':id', $id, PDO::PARAM_INT);
    $stmt->execute();
    $count = $stmt->fetchColumn();

    if ($count == 0) {
        echo "The specified book ID was not found.";
        // Rollback the transaction
        $dbh->rollBack();
        exit;
    }

    // Delete the record
    $sql = 'DELETE FROM books WHERE id = :id';
    $stmt = $dbh->prepare($sql);
    $stmt->bindValue(':id', $id, PDO::PARAM_INT);
    $stmt->execute();

    // Commit the transaction
    $dbh->commit();

    // Log success
    error_log("Delete successful for Book id: " . $_POST['id'] . " ,User ID: " . $_SESSION['username']);
    echo "Book information has been deleted.<br>";
    echo "<a href='index.php'>Return to the list</a>";
} catch (PDOException $e) {
    // Rollback in case of error
    if ($dbh->inTransaction()) {
        $dbh->rollBack();
    }
    echo "An error occurred: " . str2html($e->getMessage()) . "<br>";

    // Rollback on any exception
    if ($dbh->inTransaction()) { // Check if a transaction is active before rollback
        $dbh->rollBack();
    }
    error_log($e->getMessage());

    exit;
}
?>

```



functions.php

```php

<?php
/**
 * Sanitize input for HTML output
 *
 * @param string $arg_input The input string to sanitize
 * @return string The sanitized string
 */
function str2html(string $arg_input) :string {
    return htmlspecialchars($arg_input, ENT_QUOTES, 'UTF-8');
}

function db_open(){
    $dsn = "mysql:host=localhost;dbname=sample_db";
    $user = "phpuser";
    $password = "dummy";
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_EMULATE_PREPARES   => false,
        PDO::MYSQL_ATTR_MULTI_STATEMENTS => false,
    ];
    $dbh = new PDO($dsn, $user, $password, $options);
    return $dbh;
}

```



