+++
date = '2025-05-05T13:34:59+09:00'
draft = false
title = 'PHP Basic Index'
+++

# 🔰 Summary of Key Concepts in PHP Basics

## Overview:

This post summarizes key topics I learned through reading a PHP beginner's book.

I plan to write separate detailed articles for individual topics.

Note: I started learning PHP with prior knowledge of Python (especially Django).

It also includes customizations I implemented, such as **delete functionality**, **authorization**, and **logging**, which were not covered in the book.

---

## 1. Natural Integration with HTML & “Blended Feel”

PHP can be embedded directly into HTML.
At first, I was confused by the feeling of *"Am I writing PHP inside HTML, or HTML inside PHP?"*

Here’s a typical example:

```php
<ul>
  <?php foreach ($books as $book): ?>
    <li><?= htmlspecialchars($book['title']) ?></li>
  <?php endforeach; ?>
</ul>
```


This seamless integration is convenient for small projects, but as codebases grow, it can lead to tangled views.
That’s why modern PHP projects often use template engines like Twig or Blade to separate logic from presentation.

## 2. Frequently Used Functions: File Operations, etc.
I wish I had compiled a cheat sheet during the learning process—so I'm planning to create one in a separate post.

Some useful functions include:

fopen() / fwrite() / fread() / fclose()

file_get_contents() / file_put_contents()

isset() / empty() / is_array() / in_array()

include() / require()

Example: Logging

```php
file_put_contents('log.txt', $message . PHP_EOL, FILE_APPEND);
```

## 3. Arrays, Associative Arrays, and Simple Algorithms
Since I already have experience with programming in other languages, I skipped the basics in the book.
Still, here’s a quick reminder of useful array functions:

foreach

array_map()

array_filter()

array_reduce()

These concepts also apply to frameworks like Laravel.

## 4. Basics of Regular Expressions
Regular expressions are helpful for string processing and validation.

Example: Email validation

```php
if (preg_match('/^[\w\-\.]+@[\w\-]+\.[\w\-\.]+$/', $email)) {
  // Valid format
}
```

preg_match(): test for pattern matches

preg_replace(): perform replacements

PHP supports powerful regex (based on Perl), though each language has its own “dialect.”

## 5. Basic Security Practices in PHP
Key security points to be aware of in PHP:

XSS protection: htmlspecialchars()

SQL injection prevention: use PDO prepared statements

Password handling: password_hash() / password_verify()

Example: Secure login

```php
if (password_verify($inputPassword, $user['hashed_password'])) {
  // Authentication successful
}
```

## 6. Implementing the Delete Function
The book I used for learning PHP didn’t cover the “Delete” part of CRUD.

So I implemented it myself using the POST method for deletions.

## 7. Authentication & Authorization
I implemented login and role-based access control using sessions.

session_start() is used to manage sessions.

Access can be controlled using values like $_SESSION['user_role'].

Example: Admin-only feature

```php
if ($_SESSION['user_role'] === 'admin') {
  // Admin dashboard
}
```

## 8. Logging Strategies
Logs aren’t just for debugging—they’re also essential for operations and audits.

Ideas I implemented:

Output logs to daily files (e.g., log_2025-05-01.txt)

Use error_log() for error-level logging

Separate error logs from standard logs

Example: Logging to daily files

```php
$logFile = 'log_' . date('Y-m-d') . '.txt';
file_put_contents($logFile, $message . PHP_EOL, FILE_APPEND);
```

# Final Thoughts:
While it’s easy to get PHP up and running with a bit of Googling, building real applications requires a more comprehensive understanding—including security, logging, and access control.

More in-depth articles on each topic will follow.