+++
date = '2026-02-06T22:51:51+09:00'
draft = false
title = 'Practical Use of itertools.product'
categories = ["python"]
+++


# Generating Combination Patterns in Python

## Problem: Generating Combinations from Adjacent Keys

Consider a scenario where you need to enumerate all possible inputs from observed keypresses on a numeric keypad, including adjacent keys.

Example: When the observed input is "1"
- Keys adjacent to "1" on the keypad: "1, 2, 4"
- Possible inputs: ['1', '2', '4']

Example: When the observed input is "21"
- Adjacent keys for "2": (1, 2, 3, 5)
- Adjacent keys for "1": (1, 2, 4)
- All combinations: ['11', '21', '31', '51', '12', '22', ...] (12 patterns)

This type of processing—"creating all combinations from choices at each position"—can be implemented concisely using itertools.product.

## What is itertools.product?

Official Documentation: https://docs.python.org/3.13/library/itertools.html

itertools.product calculates the "Cartesian product."
While that sounds complex, it simply means **"all combinations of taking one element from each of multiple lists"**.

### Simple Examples
```python
from itertools import product

# All combinations of two lists
list(product(['A', 'B'], ['1', '2']))
# [('A', '1'), ('A', '2'), ('B', '1'), ('B', '2')]

# All combinations of three lists
list(product(['X', 'Y'], ['1', '2'], ['α', 'β']))
# [('X', '1', 'α'), ('X', '1', 'β'), ('X', '2', 'α'), ...]
```

### **Observed PIN Sample Code**
```python
from itertools import product

def get_pins(observed):
    # Mapping of each digit to its adjacent keys
    lock = {
        '1': ('1', '2', '4'),
        '2': ('1', '2', '3', '5'),
        '3': ('2', '3', '6'),
        '4': ('1', '4', '5', '7'),
        '5': ('2', '4', '5', '6', '8'),
        '6': ('3', '5', '6', '9'),
        '7': ('4', '7', '8'),
        '8': ('5', '7', '8', '9', '0'),
        '9': ('6', '8', '9'),
        '0': ('8', '0')
    }

    # Create a list of possible digits for each observed digit
    # Example: '21' → [('1','2','3','5'), ('1','2','4')]
    possible_digits = [lock[digit] for digit in observed]

    # Generate all combinations with product and join to strings
    # *possible_digits unpacks the list to pass as arguments
    combinations = [''.join(combo) for combo in product(*possible_digits)]

    return combinations

# Test
print(get_pins('1'))
# ['1', '2', '4']

print(get_pins('21'))
# ['11', '21', '31', '51', '12', '22', '32', '52', '13', '23', '33', '53', '14', '24', '34', '54']
# 4 × 3 = 12 patterns
```

## itertools.product vs Recursion: Which to Use?

| Aspect | itertools.product | Recursive Version |
|--------|-------------------|-------------------|
| Code Length | Short (about 3 lines) | Somewhat longer |
| Readability | Intent is clear | Requires understanding logic |
| Recommended for Production | ⭐⭐⭐ Recommended | ⭕ Good for learning |
| Library Dependency | itertools (standard) | None |

**Conclusion: Use itertools.product in production**
- No additional installation needed (standard library)
- Short code, less prone to bugs
- Good example of avoiding "reinventing the wheel"

## Reference: Recursive Implementation (For Learning)

To deepen understanding of the algorithm, let's also implement it recursively.

**Approach:**
1. For each possibility of the first digit
2. Recursively get combinations of remaining digits
3. Combine and return
```python
def get_pins(observed):
    lock = {
        '1': ('1', '2', '4'),
        '2': ('1', '2', '3', '5'),
        # ... (omitted)
    }

    # Base case: if observed is empty, return list with empty string
    if not observed:
        return ['']

    # Split into first digit and rest
    first_digit = observed[0]
    rest = observed[1:]

    result = []
    # For each possibility of the first digit
    for char in lock[first_digit]:
        # Recursively get combinations of remaining digits
        for combination in get_pins(rest):
            result.append(char + combination)

    return result
```

**Operation Image (for '21'):**
```
get_pins('21')
├─ first='2', rest='1'
│  ├─ char='1': get_pins('1') → ['1','2','4'] → ['11','12','14']
│  ├─ char='2': get_pins('1') → ['1','2','4'] → ['21','22','24']
│  ├─ char='3': get_pins('1') → ['1','2','4'] → ['31','32','34']
│  └─ char='5': get_pins('1') → ['1','2','4'] → ['51','52','54']
```

## Learning Points

### Three Essential itertools Functions to Remember
1. **product**: All combinations (Cartesian product) ← Today's topic
2. **combinations**: Unordered combinations (e.g., choosing 2 from ABC → AB, AC, BC)
3. **permutations**: Permutations (e.g., rearranging ABC → ABC, ACB, BAC, ...)

### Practical Use Cases
- Generating input candidates
- Comprehensive test case coverage
- Implementing fuzzy search
- Enumerating password candidates (security analysis)