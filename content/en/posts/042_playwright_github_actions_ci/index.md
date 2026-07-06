+++
date = '2026-07-06T14:43:19+09:00'
draft = false
title = 'Running Playwright E2E Tests with GitHub Actions (React + FastAPI + SQLite)'
+++



In my previous Playwright articles, I introduced how to create E2E tests and how to integrate GitHub Actions with a simple frontend project.

This time, I extended the workflow to a more practical setup by running Playwright against my EventLens application, which consists of:

* React (Vite)
* FastAPI
* SQLite
* Playwright
* GitHub Actions

The goal was to execute the complete E2E test suite automatically whenever code is pushed to GitHub.

---

## Project Structure

```
eventlens_calendar_poc/
├── app/
├── data/
├── frontend/
│   ├── package.json
│   ├── playwright.config.ts
│   └── tests/
├── requirements.txt
└── .github/
    └── workflows/
        └── playwright.yml
```

The frontend communicates with the FastAPI backend through Vite's proxy configuration.

---

## CI Workflow

The workflow performs the following steps:

1. Checkout the repository
2. Set up Node.js
3. Install frontend dependencies
4. Set up Python
5. Install backend dependencies
6. Initialize the SQLite database
7. Start FastAPI
8. Install Playwright browsers
9. Run Playwright tests
10. Upload the Playwright HTML report

Conceptually, the pipeline looks like this:

```
Git Push
    │
    ▼
Checkout
    │
    ▼
Setup Node.js
    │
    ▼
npm ci
    │
    ▼
Setup Python
    │
    ▼
pip install
    │
    ▼
Initialize SQLite
    │
    ▼
Start FastAPI
    │
    ▼
Install Playwright Browsers
    │
    ▼
Playwright Test
    │
    ▼
Upload HTML Report
```

---

## Setting up Node.js

The frontend uses the Node.js version defined in `.nvmrc`.

```yaml
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version-file: frontend/.nvmrc
```

Using `.nvmrc` keeps the local development environment and GitHub Actions consistent.

---

## Setting up Python

The backend uses FastAPI, so Python must also be configured.

```yaml
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.10"

- name: Install Python dependencies
  run: pip install -r requirements.txt
```

---

## Initializing SQLite

The SQLite database itself is not committed to Git.

Instead, GitHub Actions recreates it every run using SQL scripts.

```yaml
- name: Initialize SQLite database
  run: |
    sqlite3 data/eventlens.db < app/db/schema.sql
    sqlite3 data/eventlens.db < data/seeds/seed.sql
```

This guarantees that every CI run starts from a clean database.

---

## Starting FastAPI

Playwright requires the backend API.

FastAPI is launched in the background before running the tests.

```yaml
- name: Start FastAPI server
  run: |
    uvicorn app.main:app --host 127.0.0.1 --port 8001 &
```


The `&` at the end of the command starts FastAPI as a background process.


Without `&`, the GitHub Actions job would wait indefinitely for the server process to exit, and the subsequent Playwright test step would never be executed.

Running FastAPI in the background allows the workflow to continue while the API remains available during the E2E tests.


---

## Installing Playwright Browsers

One point that initially confused me was the Playwright installation process.

Running `npm ci` installs the Playwright package (`@playwright/test`), but **it does not install the browser binaries**.

Those are installed separately.

```yaml
- name: Install Playwright browsers
  working-directory: frontend
  run: npx playwright install --with-deps
```

This distinction is important for understanding Playwright in CI/CD environments.

---

## Running E2E Tests

Once everything is ready, running the tests is straightforward.

```yaml
- name: Run Playwright tests
  working-directory: frontend
  run: npm run test:e2e
```

The workflow successfully executed all four tests.

```
Running 4 tests using 2 workers

✓ loads and shows seeded events
✓ filters CME only
✓ search keyword Oncology
✓ pagination

4 passed
```

---

## Uploading the HTML Report

Finally, the Playwright HTML report is uploaded as a GitHub Actions Artifact.

```yaml
- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report
    path: frontend/playwright-report/
    retention-days: 14
```

The report can be downloaded directly from the GitHub Actions page even after the runner has been deleted.

---

## What I Learned

This project helped me understand that running Playwright in CI involves several distinct layers:

* Node.js runtime
* npm dependencies
* Playwright library
* Browser binaries
* Backend service
* Database initialization

At first, I assumed installing Playwright meant running `npm install`.

However, the browser installation is a separate step, which explains why the official Playwright Docker image already includes browser binaries.

Understanding this separation made the overall CI pipeline much clearer.

---

## Next Step

In a future article, I'd like to simplify this workflow by using the official Playwright Docker image.

Since the Docker image already contains the browser binaries and required Linux dependencies, the workflow becomes smaller and easier to maintain.

That will also be a good opportunity to compare the standard GitHub Actions runner with the Playwright Docker-based approach.
