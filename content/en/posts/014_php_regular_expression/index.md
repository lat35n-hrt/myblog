+++
date = '2025-05-12T17:23:51+09:00'
draft = false
title = 'PHP Regular Expression'
categories = ["PHP"]
+++



In PHP, validating postal codes (e.g., `123-4567`) is simple using regular expressions. This article explains how to do it and includes a summary of commonly used regex symbols and patterns.

## ✅ Regular Expression for Postal Code Format

```php
$pattern = '/^\d{3}-\d{4}$/u';

```

This regex checks for an exact match of the format "3 digits + hyphen + 4 digits".

Explanation of each symbol:

| Symbol  | Meaning                                                             |
| ------- | ------------------------------------------------------------------- |
| `/.../` | Delimiters that define the regular expression                       |
| `^`     | Anchors the match at the beginning of the string                    |
| `\d`    | Matches a single digit (0–9)                                        |
| `{3}`   | Repeats the preceding pattern (here, `\d`) exactly 3 times          |
| `-`     | Matches a literal hyphen                                            |
| `{4}`   | Repeats the preceding pattern (here, `\d`) exactly 4 times          |
| `$`     | Anchors the match at the end of the string                          |
| `u`     | Modifier to enable UTF-8 mode (necessary for multi-byte characters) |



💡 Example Code:

```php
$zipcode = '123-4567';

if (preg_match('/^\d{3}-\d{4}$/u', $zipcode)) {
    echo "Valid postal code.";
} else {
    echo "Invalid postal code.";
}

```


## 📘 Common Regex Patterns

| Pattern   | Description                                                        | Example                                                                                    |
| --------- | ------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| `.`       | Matches any single character (except newline)                      | `a.b` matches `acb`, `arb`                                                                 |
| `^`       | Matches the start of a line                                        | `^abc` matches `abc123`                                                                    |
| `$`       | Matches the end of a line                                          | `abc$` matches `123abc`                                                                    |
| `*`       | Matches 0 or more of the preceding character                       | `a*` matches `""`, `"a"`, `"aaa"`                                                          |
| `+`       | Matches 1 or more of the preceding character                       | `a+` matches `"a"`, `"aaa"`                                                                |
| `?`       | Matches 0 or 1 of the preceding character                          | `a?` matches `"a"` or `""`                                                                 |
| `{n}`     | Matches exactly n repetitions                                      | `\d{4}` matches `1234`                                                                     |
| `{n,}`    | Matches n or more repetitions                                      | `a{2,}` matches `"aa"`, `"aaa"`                                                            |
| `{n,m}`   | Matches between n and m repetitions                                | `a{2,4}` matches `"aa"`, `"aaa"`                                                           |
| `[abc]`   | Matches any one of `a`, `b`, or `c`                                | Matches `a`, `b`, or `c`                                                                   |
| `[^abc]`  | Matches any character except `a`, `b`, or `c`                      | Matches `d`, `x`, etc.                                                                     |
| `[0-9]`   | Matches any digit between 0 and 9                                  | Matches `5`                                                                                |
| `[a-z]`   | Matches any lowercase letter                                       | Matches `m`                                                                                |
| `[A-Z]`   | Matches any uppercase letter                                       | Matches `Z`                                                                                |
| `\d`      | Matches a digit (same as `[0-9]`)                                  | Matches `7`                                                                                |
| `\D`      | Matches a non-digit character (same as `[^0-9]`)                   | Matches `a`, `-`, etc.                                                                     |
| `\w`      | Matches a word character (`[a-zA-Z0-9_]`)                          | Matches `a`, `_`, `9`                                                                      |
| `\W`      | Matches a non-word character                                       | Matches `-`, `@`                                                                           |
| `\s`      | Matches a whitespace character                                     | Matches space, tab (`\t`), newline (`\n`)                                                  |
| `\S`      | Matches a non-whitespace character                                 | Matches `a`, `1`                                                                           |
| `\|`      | OR condition (matches either side)<br>Usually used with groups | `abc\|xyz` matches `abc` or `xyz` <br> `(dog\|cat)s?` matches `dog`, `dogs`, `cat`, `cats` |
| `( ... )` | Groups expressions (useful for repetition, etc.)                   | `(ab)+` matches `abab`                                                                     |
| `(?i)`    | Case-insensitive mode                                              | `(?i)abc` matches `abc`, `ABC`                                                             |
| `(?=...)` | Positive lookahead (matches only if followed by `...`)         | `\d(?=euros)` matches the 0 in "100euros" because it is immediately followed by "euros".           |
| `(?!...)` | Negative lookahead (matches only if **not** followed by `...`) | `\d(?!dollar)` matches the `0` in `"100euros"` (because it is **not** followed by `dollar`.) |




🧩 Common Modifiers

| Modifier | Description                                         |
| -------- | --------------------------------------------------- |
| `i`      | Case-insensitive matching                           |
| `m`      | Multi-line mode; `^` and `$` match line starts/ends |
| `s`      | Dot (`.`) matches newline characters                |
| `u`      | Enables UTF-8 support for multibyte characters      |
| `x`      | Ignore whitespace and allow comments in the pattern |

