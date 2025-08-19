+++
date = '2025-08-19T20:49:38+09:00'
draft = false
title = 'Understanding the Uvicorn Command by Breaking It Down'
categories = ["fastapi"]
+++


A commonly used command to start ASGI applications like FastAPI is:

```bash
uvicorn app.main:app --reload
```

At first glance, it looks simple, but it's easier to understand if you break it into two parts.

---

## ① The app.main Part

This represents the Python module path.

- Treat the app/ directory as a package

- Specify the main.py file inside it

👉 In other words, it's the module path converted from the location of main.py relative to the project root.

For example:

```
project-root/
 └── app/
      └── main.py
```


You would write it as app.main.

---

## ② The app After the Colon

This refers to the variable name of the ASGI application defined in main.py.

For example, inside main.py you might have:

```python
from fastapi import FastAPI

app = FastAPI()
```

Uvicorn loads the object named app as the ASGI application.

---

## Putting It Together

uvicorn app.main:app essentially means:

- app.main = Load the module project-root/app/main.py

- :app = Use the app object inside that module

---

## If You Change the Variable Name

If main.py is written like this:

```python
from fastapi import FastAPI

application = FastAPI()
```


Then the Uvicorn command must be:

```bash
uvicorn app.main:application --reload
```


In other words, the part after the colon must match the ASGI application variable name.

---

## Summary

- app.main = Module location (app/main.py)

- :app = The ASGI application object name inside that module

Understanding this makes the Uvicorn startup command much clearer.