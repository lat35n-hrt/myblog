+++
date = '2025-05-06T20:23:18+09:00'
draft = false
title = 'PHP array vs Python list'
+++

# Background
Basic programming books often skip exercises on PHP arrays and quickly move on to web development topics. However, this article compares PHP arrays with Python structures where relevant, focusing on areas that raised questions during learning.

# Objective
To understand the similarities and differences between PHP arrays and Python lists (and dictionaries).

# 1. PHP Arrays Have Both List and Dict Characteristics

When I first looked into how arrays are used in PHP, I was surprised to see that they were often used like dictionaries — with key-value pairs — right from the beginning.
It gave me the impression that PHP doesn’t really distinguish between list-like and dictionary-like usage.

In contrast, Python clearly separates lists and dictionaries, so if you're not familiar with Python, this comparison might initially feel more confusing than helpful.
However, the goal here is to use the comparison as a learning aid and personal reference.


# 2. Comparing Mutability: PHP vs Python

## 2.1 Mutability in Python

| Data Type              | Mutable / Immutable | Notes                               |
|------------------------|---------------------|--------------------------------------|
| `int`, `str`, `tuple`  | Immutable            | Values cannot be changed directly    |
| `list`, `dict`, `set`  | Mutable              | Values can be modified in-place      |

✅ Example 1: List (mutable)

```python
a = [1, 2, 3]
b = a  # Both refer to the same list
b.append(4)

print("a:", a)  # → a: [1, 2, 3, 4]
print("b:", b)  # → b: [1, 2, 3, 4]
```

➡️ a and b refer to the same list object. Since lists are mutable, b.append() also affects a.

→ To avoid this, use a copy:

```python
a = [1, 2, 3]
b = a.copy()  # Shallow copy
b.append(4)

print("a:", a)  # → [1, 2, 3]
print("b:", b)  # → [1, 2, 3, 4]
```

✅ Example 2: Integer (immutable)


```python
a = 10
b = a
b += 1

print("a:", a)  # → 10
print("b:", b)  # → 11
```

➡️ int is immutable. b += 1 creates a new integer object and reassigns it to b.

✅ Example 3: Object (mutable)

```python
class User:
    def __init__(self, name):
        self.name = name

a = User("Alice")
b = a
b.name = "Bob"

print("a.name:", a.name)  # → Bob
```

➡️ User objects are mutable. a and b refer to the same instance.


## 2.2 PHP Behavior: Copy-on-Write and References
PHP uses pass-by-value by default, but arrays and objects have some nuances.

● Arrays are copied, but with special behavior:

```php
$a = [1, 2, 3];
$b = $a;
$b[] = 4;
print_r($a);  // → [1, 2, 3] ← $a is unchanged
```

This may look immutable, but PHP uses a copy-on-write mechanism. Even if $a and $b initially share memory, a copy is made automatically when a modification is attempted.

● Objects are passed by reference:

```php
class User {
    public $name;
}
$a = new User();
$a->name = "Alice";
$b = $a;
$b->name = "Bob";
echo $a->name;  // → "Bob"
```

So, PHP arrays behave as if they are immutable at times, but objects are clearly mutable due to reference behavior.


## 2.3 Note: Explicit References Using &

```php
$a = [1, 2];
$b =& $a;  // Reference assignment
$b[] = 3;
print_r($a);  // → [1, 2, 3]
```


# 3. Summary of Differences

| Feature        | Python                            | PHP                                          |
|----------------|------------------------------------|-----------------------------------------------|
| `list`         | Mutable                            | Arrays are mutable, but copied on write       |
| `dict`         | Mutable                            | Associative arrays serve a similar purpose    |
| `str`          | Immutable                          | Strings are also immutable                    |
| Objects        | Mutable                            | Passed by reference, hence mutable            |
| Special Traits | Explicit with `copy()` or slicing  | Copy-on-write, reference (`&`) available      |



# 4. Conclusion
Python makes mutability an explicit part of its design, and developers often consider it directly.

PHP, while defaulting to value passing, behaves more like a mutable environment under the hood — especially with copy-on-write and reference mechanisms.

These differences become particularly important when designing algorithms or managing shared objects.




