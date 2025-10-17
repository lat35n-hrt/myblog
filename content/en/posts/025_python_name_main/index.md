+++
date = '2025-10-17T21:17:08+09:00'
draft = false
title = 'Basics of if __name__ == "__main__":'
categories = ["python"]
+++


In Python, it’s common to place code that should only run when the file is executed directly inside the

if __name__ == "__main__": block.

```python
def fetch_articles(debug=False):
    if debug:
        print("Debug mode: fetching mock articles...")
        return [{"title": "Test Article"}]
    # Actual implementation ...
    pass


if __name__ == "__main__":
    articles = fetch_articles(debug=True)
    print(articles)
```

💡 When to Use It

When you want to debug a script by running it directly

When you don’t want extra code to run when the file is imported as a module

When you want to combine standalone execution and code reusability

## 🧭 Behavior Summary

| How You Run It     | Value of `__name__` | Is the Block Executed? |
| ------------------ | ------------------- | ---------------------- |
| `python script.py` | `"__main__"`        | ✅ Yes                  |
| `import script`    | `"script"`          | ❌ No                   |



## 🧠 Practical Use

During development, it’s useful to include a simple test or debug entry point like this:

```python
if __name__ == "__main__":
    # Run quick tests or debug code
    articles = fetch_articles(debug=True)
```

You can confirm behavior easily with debug=True

This section won’t run when using pytest or importing the module, so it’s safe to leave in

## ✏️ Summary

if __name__ == "__main__": is a standard Python idiom
for separating code that should only run when a file is executed directly.
It’s a handy entry point for debugging and simple standalone testing.