+++
date = '2026-02-05T15:33:23+09:00'
draft = false
title = 'String Cleaning with Python Regular Expressions'
categories = ["python"]
+++


Text data preprocessing frequently involves removing unwanted characters and pattern matching. This article introduces practical patterns using Python's `re` module.

## 1. re.sub() - String Replacement and Cleaning

### Basic Syntax
```python
import re
re.sub(pattern, replacement, string)
```

- **pattern**: Regular expression pattern to search for
- **replacement**: String to replace with
- **string**: Target string

### Practical Example: Remove All Except Specific Characters
```python
text = "hello, world! don't stop #learning"
cleaned = re.sub(r"[^a-zA-Z0-9'\s]", ' ', text)
print(cleaned)
# "hello  world  don't stop  learning"
```

### Pattern Explanation
```
r"[^a-zA-Z0-9'\s]"

r""         : raw string (backslashes not escaped)
[]          : character class (defines a set of characters)
^           : negation (when used at the start within [])
a-zA-Z      : all alphabetic characters
0-9         : all digits
'           : apostrophe (literal)
\s          : whitespace characters (space, tab, newline, etc.)

→ Replace "anything except alphanumeric, apostrophe, and whitespace" with space
```

### Use Cases
- User input sanitization
- Filename normalization
- Text mining preprocessing
- Log data cleaning

---

## 2. re.search() - Pattern Search

### Basic Syntax
```python
import re
re.search(pattern, string)
```

Returns a `Match` object if a match is found, otherwise returns `None`.

### Practical Example: Check for Alphanumeric Characters
```python
# Check if word contains alphanumeric characters
word = "don't"
if re.search(r'[a-zA-Z0-9]', word):
    print("Valid word")
else:
    print("Invalid word (symbols only)")
```

### Advanced: Filter Out Apostrophe-Only Words
```python
words = ["hello", "world", "'", "don't", "''"]
valid_words = [w for w in words if re.search(r'[a-zA-Z0-9]', w)]
print(valid_words)
# ['hello', 'world', "don't"]
```

### Use Cases
- Input validation
- Filtering operations
- Conditional logic
- Data cleansing

---

## 3. Combining in Practice

### Text Analysis Pipeline
```python
import re

def clean_and_filter(text):
    """Clean text and extract only valid words"""
    # 1. Remove special characters
    cleaned = re.sub(r"[^a-zA-Z0-9'\s]", ' ', text)

    # 2. Split into words
    words = cleaned.split()

    # 3. Filter only words containing alphanumeric characters
    valid_words = [w for w in words if re.search(r'[a-zA-Z0-9]', w)]

    return valid_words

# Usage example
text = "hello, world! don't stop. ''' #hashtag"
result = clean_and_filter(text)
print(result)
# ['hello', 'world', "don't", 'stop', 'hashtag']
```

### Log File Analysis
```python
def extract_error_lines(log_text):
    """Extract only error lines"""
    lines = log_text.split('\n')
    error_lines = [line for line in lines
                   if re.search(r'ERROR|CRITICAL', line)]
    return error_lines
```

---

## 4. Common Regular Expression Patterns

| Pattern | Meaning | Use Case |
|---------|---------|----------|
| `[^...]` | Negation (anything except ...) | Remove unwanted characters |
| `\s` | Whitespace characters | Space, tab, newline |
| `\d` | Digits (0-9) | Detect numbers |
| `\w` | Word characters (a-zA-Z0-9_) | Identifier validation |
| `+` | One or more repetitions | Consecutive patterns |
| `*` | Zero or more repetitions | Optional patterns |

---

## Summary

- **re.sub()**: Used for string replacement and cleaning
- **re.search()**: Used for pattern existence checking
- **Combination**: Enables building practical text processing pipelines

It's easy to handle string operations case-by-case, but regular expressions often provide simpler solutions.

---

## References
- [Python re Module Official Documentation](https://docs.python.org/3/library/re.html)
- [Regular Expression Testing Tool regex101](https://regex101.com/)