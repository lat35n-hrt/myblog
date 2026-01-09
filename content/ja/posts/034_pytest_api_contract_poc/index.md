+++
date = '2026-01-09T17:12:09+09:00'
draft = false
title = 'pytest（API）で FastAPI の Smoke/Contract テストを作る（ASGITransport + anyio）'
categories = ["pytest"]
+++


## はじめに

FastAPI の最小 API（`/health`, `/api/items`）に対して、pytest で **スモーク／コントラクト検証**を自動化します。

本記事では、実際に試行錯誤しながら構築したプロセスを作業ログ的に記録しています。

### 実現したいこと

- `pytest -q` でローカル実行できる
- 外部 I/O（ネットワーク、外部DB）に依存しない
- 失敗時のメッセージが読みやすい（assert に意図を残す）
- CI にそのまま載せられる（= 余計な起動手順がいらない）

### 検証環境

- OS: macOS **13.7.2**
- Python: **3.10.11**
- pytest: **9.0.2**
- httpx: **0.28.1**
- FastAPI: **0.128.0**
- Playwright: **1.57.0**（同一リポジトリ内の E2E 用）
- anyio: **4.x**（CI では 4.12.1 が入っていました）

確認コマンドは以下です。

```bash
cd backend
python -V
python -m pip show fastapi pytest httpx anyio
python -m pytest --help | grep -i anyio
```

---

## 方式の選定：uvicorn 起動はしない（プロセス内で FastAPI を叩く）

今回のポイントは、**httpx の ASGITransport**（または AsyncClient）を使い、FastAPI をプロセス内で直接呼ぶことです。

### メリット

- CI が軽い
- 起動待ちが不要
- flaky を増やさない

### デメリット

- 本当の HTTP レイヤ（ネットワーク、プロキシ等）の検証ではない

ただし、スモーク／コントラクト検証には十分です。

---

## 前提：backend を作業ルートに固定する

今回の構成では、API テストは `backend/tests` に配置します。import の解決も含めて、**普段は `cd backend` で pytest を回す**運用に固定します。

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

## pytest.ini：探索範囲を固定して事故を防ぐ

`backend/pytest.ini` は以下のように設定します。

```ini
[pytest]
testpaths = tests
norecursedirs = e2e
addopts = -q
markers =
    anyio: run async tests via anyio
```

### 設定の意図

- `testpaths = tests`：pytest は `backend/tests` だけ探索する
- `norecursedirs = e2e`：Playwright 側資産に触れない（混線防止）
- `addopts = -q`：出力最小化（CI 向け）

---

## アプリの import を環境変数で切り替える（FASTAPI_APP）

FastAPI アプリを「どのモジュールから読むか」を固定値にすると、PoC の再利用性が落ちます。そこで `FASTAPI_APP` を使います。

### 使用例

```bash
FASTAPI_APP="app.main:app"
```

`backend/tests/app_import.py` はこの環境変数を参照し、`app.main` を import して `app` を取り出します。

この"環境変数でアプリを特定する"設計にしておくと、将来 PoC のアプリ構成が変わってもテスト側の変更は最小で済みます。

---

## conftest.py：fixture を集約する（app / client）

pytest は `conftest.py` を自動で読み込み、fixture を登録できます。今回の最小構成は `app` と `client` です。

### 同期でやる場合の注意（今回ハマった点）

httpx の `ASGITransport` は（バージョンによって）**同期 Client のコンテキストマネージャに非対応**になり、`__enter__` が無いエラーに当たることがあります。

その場合、**AsyncClient + anyio** に寄せるのが最も安定します。

### 例：AsyncClient を使う fixture（推奨）

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

このとき、テスト側は `@pytest.mark.anyio` を付ける（またはモジュールに付ける）必要があります。

---

## テスト：/health（最小スモーク）

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

### ポイント

- "何が期待で何が実際か" を assert メッセージに残す
- `body={r.text}` を入れると CI ログで原因が即わかる（特に 404/500 時）

---

## テスト：/api/items（GET と POST のコントラクト確認）

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

## FastAPI 側：POST を最小で揃える（コントラクトを満たす）

pytest が "正しい" というより、**API の仕様が揃っていないと contract テストは必ず落ちます**。

今回の収束点（最小 echo）は以下です。

```python
# backend/app/main.py
from fastapi import Body

@app.post("/api/items", status_code=201)
async def create_item(payload: dict = Body(...)):
    return payload
```

ここで一度 `405 Method Not Allowed` を踏むのは自然な流れです（POST が無いので）。テストが "仕様の穴" を検知できている証拠になります。

---

## 実行手順（普段は cd backend 前提）

```bash
cd backend
source .venv_api/bin/activate
FASTAPI_APP="app.main:app" python -m pytest -q
```

結果：
```
          [100%]
3 passed in 0.85s

```

---

## つまずきポイント（今回のデバッグログ）

この PoC は、以下の "典型的な詰まりどころ" を一通り踏んで解消しています。

### 1) `fixture 'client' not found`

- **原因**：`conftest.py` が探索されていない（rootdir が想定とズレる等）
- **対策**：`cd backend` で pytest を実行し、`backend/pytest.ini` を確実に効かせる

### 2) `ModuleNotFoundError: No module named 'fastapi'`

- **原因**：API 用 venv に `requirements.txt`（fastapi）を入れていない
- **対策**：API venv は `requirements.txt + requirements-dev-api.txt` を入れる

### 3) `ASGITransport has no __enter__`

- **原因**：同期 `httpx.Client` で ASGITransport を使うとバージョン差で落ちることがある
- **対策**：`AsyncClient` + `pytest.mark.anyio` に寄せる

### 4) `PytestRemovedIn9Warning: async fixture ... no plugin`

- **原因**：async fixture を使っているのに anyio 等が有効でない
- **対策**：`anyio>=4` を依存に明示し、テストに `@pytest.mark.anyio` を付与

### 5) `POST /api/items => 405`

- **原因**：API 実装が GET のみ
- **対策**：FastAPI に POST を最小実装で追加し contract を満たす

---

## まとめ

- httpx（ASGITransport）で FastAPI をプロセス内呼び出しし、CI を軽量にしました
- `FASTAPI_APP` でアプリの import を切り替え可能にしました
- async fixture + anyio に寄せることで、httpx の transport 差異を吸収しました
- テストが「仕様の穴（POST なし）」を検知し、実装を収束させました

この構成により、外部サーバーを起動することなく、高速で安定したAPIテストが実現できます。