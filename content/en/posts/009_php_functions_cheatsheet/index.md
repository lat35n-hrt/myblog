+++
date = '2025-05-06T08:03:02+09:00'
draft = false
title = 'PHP Function Cheat Sheet (For Beginners)'
categories = ["PHP"]
+++

# PHP Function Cheat Sheet

This cheat sheet was created from a beginner's perspective to keep a handy list of commonly used PHP functions.

| Function / Variable               | Category         | Description / Notes |
|----------------------------------|------------------|---------------------|
| `htmlspecialchars()`             | XSS Protection    | Escapes HTML for safe output |
| `password_hash()`                | Security (Auth)   | Hashes a password securely |
| `password_verify()`              | Security (Auth)   | Verifies user input against a hash |
| `bin2hex(random_bytes(32))`      | CSRF Protection   | Generates a cryptographically secure token |
| `hash_equals($a, $b)`            | CSRF Protection   | Constant-time comparison to prevent timing attacks |
| `session_start()`                | Session           | Starts a session |
| `$_SESSION`                      | Session           | Session variable (associative array) |
| `session_destroy()`              | Session           | Completely destroys the session |
| `session_regenerate_id()`        | Session           | Regenerates session ID (prevents fixation attacks) |
| `session_unset()`                | Session           | Clears all session variables |
| `ini_set('session.cookie_httponly', true)` | Session | Prevents JavaScript access to session cookie (XSS protection) |
| `header()`                       | HTTP Handling     | Used for redirection or setting HTTP headers |
| `strip_tags()`                   | Input Handling    | Removes HTML tags from input |
| `filter_input()`                 | Input Handling    | Retrieves and filters external input |
| `fopen()`                        | File Handling     | Opens a file with specified mode |
| `fread()`                        | File Handling     | Reads from a file |
| `fwrite()`                       | File Handling     | Writes to a file |
| `fclose()`                       | File Handling     | Closes a file |
| `file_get_contents()`            | File Handling     | Reads entire file into a string |
| `file_put_contents()`            | File Handling     | Writes entire string to a file (simple method) |
| `file_exists()`                  | File Handling     | Checks if a file exists |
| `unlink()`                       | File Handling     | Deletes a file |
| `is_readable()`                  | File Handling     | Checks if file is readable |
| `is_writable()`                  | File Handling     | Checks if file is writable |
| `strlen()`                       | String Handling   | Gets the length of a string |
| `str_replace()`                  | String Handling   | Replaces part of a string |
| `strpos()`                       | String Handling   | Finds the position of a substring |
| `substr()`                       | String Handling   | Extracts part of a string |
| `trim()`                         | String Handling   | Removes whitespace and line breaks from ends |
| `explode()`                      | Array Handling    | Splits string into an array |
| `implode()`                      | Array Handling    | Joins array elements into a string |
| `array_merge()`                  | Array Handling    | Merges arrays |
| `array_map()`                    | Array Handling    | Applies a function to each element of an array |
| `array_filter()`                 | Array Handling    | Filters array elements by condition |
| `array_slice()`                  | Array Handling    | Extracts a portion of an array |
| `in_array()`                     | Array Handling    | Checks if a value exists in an array |
| `isset()`                        | Variable Check    | Checks if a variable is set |
| `empty()`                        | Variable Check    | Checks if a variable is empty |
| `is_array()`                     | Type Check        | Checks if a variable is an array |
| `is_null()`                      | Type Check        | Checks if a variable is NULL |
| `json_encode()`                  | JSON Handling     | Converts array or object to JSON |
| `json_decode()`                  | JSON Handling     | Parses JSON to array or object |
| `time()` / `date()`              | DateTime          | Gets current timestamp or formats a date |
| `print_r()`                      | Debugging         | Prints human-readable info about a variable |
| `var_dump()`                     | Debugging         | Dumps detailed type and value info |

## Superglobals

| Function / Variable               | Category          | Description / Notes |
|----------------------------------|-------------------|---------------------|
| `$_GET`                          | User Input        | Gets URL parameters |
| `$_POST`                         | User Input        | Gets POST form values |
| `$_REQUEST`                      | User Input        | Combines GET/POST/COOKIE (not recommended) |
| `$_SERVER`                       | Server Variables  | Gets request and server info |
| `$_COOKIE`                       | Session / Cookie  | Gets cookies sent by the user |