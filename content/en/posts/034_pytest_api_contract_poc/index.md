+++
date = '2026-01-09T17:12:16+09:00'
draft = false
title = 'Creating FastAPI Smoke/Contract Tests with pytest (ASGITransport + anyio)'
categories = ["pytest"]
+++


## Introduction

We will automate **smoke/contract testing** with pytest for a minimal FastAPI (`/health`, `/api/items`).

This article documents the trial-and-error process in a work log style.

### Goals

- Can be executed locally with `pytest -q`
- No dependency on external I/O (network, external DB)
- Readable failure messages (preserve intent in assertions)
- Can be deployed directly to CI (no extra startup procedures needed)

### Testing Environment

- OS: macOS **13.7.2**
- Python: **3.10.11**
- pytest: **9.0.2**
- httpx: **0.28.1**
- FastAPI: **0.128.0**
- Playwright: **1.57.0** (for E2E in the same repository)
- anyio: **4.x** (4.12.1 was installed in CI)

Verification commands:

```bash
cd backend
python -V
python -m pip show fastapi pytest httpx anyio
python -m pytest --help | grep -i anyio
```

---

## Approach: Don't Start uvicorn (Call FastAPI In-Process)

The key point is to use **httpx's ASGITransport** (or AsyncClient) to call FastAPI directly in-process.

### Advantages

- Lightweight CI
- No startup waiting time
- Doesn't increase flakiness

### Disadvantages

- Not testing the real HTTP layer (network, proxy, etc.)

However, this is sufficient for smoke/contract testing.

---

## Prerequisite: Fix backend as the Working Root

In this configuration, API tests are placed in `backend/tests`. Including import resolution, we standardize on **running pytest with `cd backend`**.

```
backend/
  app/main.py
  tests/
    conftest.py
    app_import.py
    test_health.py
    test_items.py
  pytest.ini
```

---

## pytest.ini: Fix Search Scope to Prevent Accidents

Configure `backend/pytest.ini` as follows:

```ini
[pytest]
testpaths = tests
norecursedirs = e2e
addopts = -q
markers =
    anyio: run async tests via anyio
```

### Configuration Intent

- `testpaths = tests`: pytest only searches `backend/tests`
- `norecursedirs = e2e`: Don't touch Playwright assets (prevent interference)
- `addopts = -q`: Minimize output (for CI)

---

## Switch App Import with Environment Variable (FASTAPI_APP)

If we hardcode "which module to read the FastAPI app from," the PoC's reusability decreases. So we use `FASTAPI_APP`.

### Usage Example

```bash
FASTAPI_APP="app.main:app"
```

`backend/tests/app_import.py` references this environment variable, imports `app.main`, and retrieves `app`.

By designing with "specify the app via environment variable," future changes to the PoC's app structure require minimal changes on the test side.

---

## conftest.py: Consolidate Fixtures (app / client)

pytest automatically loads `conftest.py` and can register fixtures. The minimal setup is `app` and `client`.

### Note on Synchronous Approach (Where I Got Stuck)

httpx's `ASGITransport` (depending on version) **doesn't support context managers for synchronous Client**, resulting in a `__enter__` missing error.

In that case, **leaning toward AsyncClient + anyio** is most stable.

### Example: Fixture Using AsyncClient (Recommended)

```python
# backend/tests/conftest.py
from __future__ import annotations

import pytest
import httpx

from tests.app_import import load_fastapi_app

@pytest.fixture(scope="session")
def app():
    return load_fastapi_app()

@pytest.fixture
async def client(app):
    transport = httpx.ASGITransport(app=app)
    async with httpx.AsyncClient(transport=transport, base_url="http://test") as c:
        yield c
```

In this case, the test side needs to add `@pytest.mark.anyio` (or add it to the module).

---

## Test: /health (Minimal Smoke)

```python
# backend/tests/test_health.py
import pytest

@pytest.mark.anyio
async def test_health_returns_200_and_json(client):
    r = await client.get("/health")
    assert r.status_code == 200, f"GET /health should return 200. got={r.status_code}, body={r.text}"
    data = r.json()
    assert isinstance(data, dict), f"/health should return JSON object. got={type(data)}"
```

### Key Points

- Leave "what is expected vs what is actual" in assert messages
- Including `body={r.text}` makes the cause immediately clear in CI logs (especially for 404/500)

---

## Test: /api/items (GET and POST Contract Verification)

```python
# backend/tests/test_items.py
import pytest

@pytest.mark.anyio
async def test_get_items_returns_200_and_json(client):
    r = await client.get("/api/items")
    assert r.status_code == 200, f"GET /api/items should return 200. got={r.status_code}, body={r.text}"
    assert isinstance(r.json(), (list, dict)), f"Expected JSON list or object. got={r.text}"

@pytest.mark.anyio
async def test_post_item_returns_200_or_201_and_json(client):
    payload = {"name": "demo", "value": 1}
    r = await client.post("/api/items", json=payload)
    assert r.status_code in (200, 201), f"POST /api/items should return 200 or 201. got={r.status_code}, body={r.text}"
    assert isinstance(r.json(), dict), f"POST response should be JSON object. got={r.text}"
```

---

## FastAPI Side: Implement POST Minimally (Satisfy the Contract)

Rather than pytest being "correct," **contract tests will always fail if the API specification isn't aligned**.

The convergence point (minimal echo):

```python
# backend/app/main.py
from fastapi import Body

@app.post("/api/items", status_code=201)
async def create_item(payload: dict = Body(...)):
    return payload
```

Encountering `405 Method Not Allowed` once is a natural flow (since POST doesn't exist), and it's proof that the test can detect "holes in the specification."

---

## Execution Steps (Assuming cd backend)

```bash
cd backend
source .venv_api/bin/activate
FASTAPI_APP="app.main:app" python -m pytest -q
```

Result:
```
          [100%]
3 passed in 0.85s

```

---

## Stumbling Points (Debug Log from This Work)

This PoC went through all of these "typical sticking points" and resolved them.

### 1) `fixture 'client' not found`

- **Cause**: `conftest.py` not being searched (rootdir deviates from expectation, etc.)
- **Solution**: Execute pytest with `cd backend` to ensure `backend/pytest.ini` takes effect

### 2) `ModuleNotFoundError: No module named 'fastapi'`

- **Cause**: API venv doesn't include `requirements.txt` (fastapi)
- **Solution**: API venv should include `requirements.txt + requirements-dev-api.txt`

### 3) `ASGITransport has no __enter__`

- **Cause**: Using ASGITransport with synchronous `httpx.Client` can fail due to version differences
- **Solution**: Lean toward `AsyncClient` + `pytest.mark.anyio`

### 4) `PytestRemovedIn9Warning: async fixture ... no plugin`

- **Cause**: Using async fixtures but anyio etc. is not enabled
- **Solution**: Explicitly declare `anyio>=4` as a dependency and add `@pytest.mark.anyio` to tests

### 5) `POST /api/items => 405`

- **Cause**: API implementation only has GET
- **Solution**: Add minimal POST implementation to FastAPI to satisfy the contract

---

## Summary

- Used httpx (ASGITransport) to call FastAPI in-process, making CI lightweight
- Made app import switchable with `FASTAPI_APP`
- Leaned toward async fixture + anyio to absorb httpx transport differences
- Tests detected "specification holes (missing POST)" and converged the implementation

With this configuration, we can achieve fast and stable API testing without starting external servers.