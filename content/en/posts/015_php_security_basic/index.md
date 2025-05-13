+++
date = '2025-05-13T09:46:18+09:00'
draft = false
title = 'PHP Security Essentials'
categories = ["PHP"]
+++


PHP and Security: 4 Essential Measures Every Developer Should Know
When developing web applications with PHP, implementing proper security measures is essential. Leaving vulnerabilities unattended can lead to serious consequences, such as data breaches or website defacement.

This article introduces four essential security practices every PHP developer should be familiar with:

- Cross-Site Scripting (XSS) Protection

- SQL Injection Prevention

- Secure Password Storage

- Cross-Site Request Forgery (CSRF) Protection

<br>

## 1. XSS Protection: htmlspecialchars()
What is XSS?
Cross-Site Scripting (XSS) is an attack in which malicious scripts are injected into web pages and executed in the user’s browser. This typically occurs when user input is directly embedded into HTML, such as in comments or message boards.

How to Prevent It
Use htmlspecialchars() in PHP to escape special characters, which prevents script execution in HTML output.


```php
$user_input = $_GET['comment'];
$safe_output = htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');
echo $safe_output;

```

Key Points
Use ENT_QUOTES as the second argument to escape both single and double quotes.

Explicitly specify the character encoding (e.g., 'UTF-8') to ensure correct behavior.

<br>

## 2. SQL Injection Prevention: Prepared Statements (PDO)
What is SQL Injection?
SQL Injection is an attack where malicious SQL code is inserted into a query through user input. This is especially dangerous in login forms or any feature that interacts with the database.

How to Prevent It
Use PHP Data Objects (PDO) with prepared statements to safely bind user input using placeholders. This protects against unintended query execution.

```php
$pdo = new PDO('mysql:host=localhost;dbname=testdb;charset=utf8', 'user', 'password');
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->bindValue(':email', $email, PDO::PARAM_STR); // Bind the value directly
$stmt->execute();

```

◆Key Points

Never concatenate raw user input into SQL statements.

Always use placeholders (? or :name) and prefer bindValue() for predictable behavior.

bindParam() passes variables by reference, which can lead to unexpected behavior in loops.

<br>

## 3. Secure Password Storage: password_hash() / password_verify()

Why Not Store Passwords in Plaintext?
Storing passwords as plain text puts users at high risk if the database is leaked. Hashing passwords helps mitigate damage in such scenarios.

How to Do It
PHP provides built-in functions for secure password hashing and verification.

At registration (hashing the password):

```php
$hash = password_hash($password, PASSWORD_DEFAULT);

```

At login (verifying the password):

```php
if (password_verify($input_password, $hash_from_db)) {
    echo "Login successful";
} else {
    echo "Login failed";
}

```

◆　Key Points

Use PASSWORD_DEFAULT to apply PHP’s recommended hashing algorithm (currently bcrypt).

password_hash() automatically handles salt generation—no need to generate it manually.

<br>

## 4. CSRF Protection: Token-Based Validation
What is CSRF?
Cross-Site Request Forgery (CSRF) is an attack that tricks a logged-in user into performing actions they didn’t intend, like clicking a “delete account” or “send money” button.

How to Prevent It
Generate a CSRF token and embed it in forms. Then, validate the token on the server when the form is submitted.

Step 1: Embed the token in the form



```php
session_start();
if (empty($_SESSION['token'])) {
    $_SESSION['token'] = bin2hex(random_bytes(32));
}
?>

<form method="post" action="submit.php">
  <input type="hidden" name="csrf_token" value="<?php echo htmlspecialchars($_SESSION['token'], ENT_QUOTES, 'UTF-8'); ?>">
  <!-- Other input fields -->
  <input type="submit" value="Submit">
</form>

```

Step 2: Validate the token on the server

```php
session_start();

if (!isset($_POST['csrf_token']) || $_POST['csrf_token'] !== $_SESSION['token']) {
    die('Invalid request');
}

// Continue with normal processing

```

◆　Key Points

Ideally, generate a unique token per session or per form.

CSRF protection is also needed in JavaScript-based forms (e.g., Ajax + Cookie-based authentication).

⚠️ Do not reuse CSRF tokens. Invalidate them after use with unset($_SESSION['token']). Also, run session_regenerate_id(true) after login to prevent session fixation attacks.

<br>

## Summary

| Threat        | Countermeasure                              |
| ------------- | ------------------------------------------- |
| XSS           | Escape output using `htmlspecialchars()`    |
| SQL Injection | Use `PDO` with prepared statements          |
| Password Leak | Use `password_hash()` / `password_verify()` |
| CSRF          | Embed and validate CSRF tokens              |
