+++
date = '2026-07-07T15:03:11+09:00'
draft = false
title = 'Playwright + GitHub Actions (Docker Edition)'
categories = ["playwright"]
+++


## Overview

In my previous article, I built an end-to-end (E2E) testing workflow using Playwright and GitHub Actions.

This time, I replaced the standard GitHub runner environment with the official Playwright Docker image and compared the differences.

The goal was not only to make the workflow work, but also to understand what the Docker image already provides and what still needs to be configured.

---

## Environment

- Frontend: React + Vite
- Node.js: 22.12.0
- Backend: FastAPI
- Python: 3.10
- Database: SQLite
- Test Framework: Playwright 1.57.0
- CI: GitHub Actions
- Container: `mcr.microsoft.com/playwright:v1.57.0-noble`

---

## Why Use the Official Playwright Docker Image?

The official Playwright image already includes:

- Node.js
- Chromium
- Firefox
- WebKit
- Required browser dependencies

Therefore, the following step from the standard workflow became unnecessary.

```yaml
- name: Install Playwright browsers
  run: npx playwright install --with-deps
```

The browser installation is already handled by the Docker image.

---

## Workflow

```yaml
container:
  image: mcr.microsoft.com/playwright:v1.57.0-noble
```

The workflow is configured to run manually.

```yaml
on:
  workflow_dispatch:
```

This allowed me to compare the Docker workflow independently from the existing CI pipeline.

---

## Problems Encountered

Building the Docker workflow was not simply a copy-and-paste exercise.

Several issues appeared during implementation.

### 1. `container:` Position

Initially I placed:

```yaml
container:
```

at the top level of the workflow.

The correct location is inside the job.

```yaml
jobs:
  e2e:
    runs-on: ubuntu-latest

    container:
      image: mcr.microsoft.com/playwright:v1.57.0-noble
```

---

### 2. Node.js Setup

My original workflow contained:

```yaml
uses: actions/setup-node@v4
```

After testing, I confirmed that the Playwright Docker image already provides Node.js.

Therefore, this step could be removed.

---

### 3. Python

My backend uses FastAPI.

Python still needs to be configured using:

```yaml
uses: actions/setup-python@v5
```

However,

```bash
pip install -r requirements.txt
```

failed.

Using

```bash
python -m pip install -r requirements.txt
```

worked correctly and is also the recommended approach in many CI environments.

---

### 4. SQLite

The Playwright Docker image does not include the SQLite CLI.

I installed it during the workflow.

```bash
apt-get update
apt-get install -y sqlite3
```

---

### 5. Playwright Version

Initially I used a newer Docker image.

Playwright reported:

```
Please update docker image as well.
```

The solution was to match the Docker image version with the Playwright package version.

```
@playwright/test 1.57
↓

mcr.microsoft.com/playwright:v1.57.0-noble
```

Keeping these versions synchronized is important.

---

## Result

The workflow completed successfully.

The Docker version now performs:

- npm install
- Python setup
- dependency installation
- SQLite initialization
- FastAPI startup
- Playwright E2E tests
- HTML report upload

without requiring browser installation.

---

## Comparison

| Standard Workflow | Docker Workflow |
|-------------------|-----------------|
| setup-node | Not required |
| Playwright browser installation | Not required |
| setup-python | Required |
| SQLite CLI | Additional installation required |
| FastAPI | Required |
| Playwright tests | Same |

---

## Lessons Learned

Using the official Playwright Docker image simplifies browser management, but it is not a complete application environment.

Backend-specific tools such as SQLite still need to be installed.

The implementation process also helped me understand the relationship between:

- GitHub Actions runners
- Docker containers
- Playwright browser binaries
- Node.js
- Python
- SQLite

rather than simply copying a sample workflow.