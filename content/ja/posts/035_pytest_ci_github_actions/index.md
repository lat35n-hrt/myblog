+++
date = '2026-01-09T17:12:38+09:00'
draft = false
title = 'FastAPI + pytest + GitHub Actions で最小CI環境を構築する'
categories = ["pytest"]
+++


## はじめに

この記事では、FastAPI で作成したAPIに対して pytest でスモークテスト・コントラクトテストを実施し、GitHub Actions で自動テストを回す最小構成を作ります。外部I/Oに依存しない、再現可能なCI環境の構築が目的です。

## ゴール

この記事を読み進めることで、以下の状態を実現できます。

- ローカル環境で `pytest` が正常に動作する
- GitHub Actions で `pip install → pytest` が自動実行される
- 外部ネットワークや外部DBに依存せず、常に再現可能なテストが回る

## 動作環境

### CI環境（GitHub Actions）

- Runner: `ubuntu-latest`
- Actions:
  - `actions/checkout@v4`
  - `actions/setup-python@v5`
- Python: `3.10`（ワークフローで固定）
- インストールされるパッケージ（CI実行時の確認値）:
  - pytest **9.0.2**
  - httpx **0.28.1**
  - anyio **4.12.1**

ワークフローでは Python バージョンを以下のように固定しています。

```yaml
- uses: actions/setup-python@v5
  with:
    python-version: "3.10"
```

## テスト対象のAPI

今回は最小構成として、以下の3つのエンドポイントをテスト対象にします。

- `GET /health` - ヘルスチェック用
- `GET /api/items` - アイテム一覧取得
- `POST /api/items` - アイテム作成（コントラクト確認用の最小エコー実装）

pytest では httpx の `ASGITransport` を使用してアプリをプロセス内で叩く構成にするため、CI環境でも uvicorn の起動は不要です。

## ディレクトリ構成

プロジェクトのディレクトリ構成は以下のようになります。**重要なポイントは、API テストの作業ディレクトリを `backend` に固定することです。**

```
playwright_flake_triage_poc/
├─ backend/
│  ├─ app/
│  │  └─ main.py
│  ├─ tests/
│  │  ├─ conftest.py
│  │  ├─ app_import.py
│  │  ├─ test_health.py
│  │  └─ test_items.py
│  ├─ pytest.ini
│  ├─ requirements.txt
│  ├─ requirements-dev-api.txt
│  └─ requirements-dev-e2e.txt
└─ .github/
   └─ workflows/
      └─ ci.yml
```

### 運用上のポイント

- **API テストは常に `cd backend` してから実行します**
- Playwright などE2E用の資産は `backend/e2e/` に配置し、pytest の探索対象外にします

## 仮想環境の分離戦略

依存関係の混在による混乱を避けるため、今回は仮想環境を2系統に分けます。

- API用: `backend/.venv_api`
- E2E用: `backend/.venv_e2e`

`.gitignore` には以下を追加しておきます。

```
backend/.venv_*/
```

## 依存関係の管理

### サーバ基本パッケージ（`backend/requirements.txt`）

FastAPI アプリケーション本体を動かすための基本パッケージです。

```
fastapi
uvicorn[standard]
jinja2
```


この実行環境では
```bash
 % python -m pip show fastapi uvicorn anyio | egrep '^(Name|Version):'
Name: fastapi
Version: 0.128.0
Name: uvicorn
Version: 0.40.0
Name: anyio
Version: 4.12.1
```

### API テスト用パッケージ（`backend/requirements-dev-api.txt`）

pytest と httpx を中心とした、API テスト実行に必要なパッケージです。

```
pytest>=8.0.0
httpx>=0.27.0
anyio>=4.0.0
```

## pytest の設定

`backend/pytest.ini` でテストの探索範囲を明示的に制限し、意図しないファイルの読み込みを防ぎます。

```ini
[pytest]
testpaths = tests
norecursedirs = e2e
addopts = -q
markers =
    anyio: run async tests via anyio
```

### 設定のポイント

- `testpaths = tests`: pytest は `backend/tests` ディレクトリのみを探索対象にします
- `norecursedirs = e2e`: Playwright 用のディレクトリを探索対象から除外し、混線を防ぎます
- `addopts = -q`: CI・ローカル両方で出力を最小化し、見やすくします

## テスト実装の構造

### FastAPI アプリのインポート（`backend/tests/app_import.py`）

FastAPI アプリケーションの import パスを環境変数 `FASTAPI_APP` で切り替え可能にしています。

設定例: `FASTAPI_APP=app.main:app`

### pytest fixture の設定（`backend/tests/conftest.py`）

- `app` fixture: FastAPI アプリケーションをロードします
- `client` fixture: httpx で ASGI アプリを直接叩くクライアントを提供します

この構成により、実際のHTTPサーバを起動することなく、APIの動作を検証できます。

## ローカルでの実行手順

### 1. 仮想環境の作成と有効化

```bash
cd backend
python3 -m venv .venv_api
source .venv_api/bin/activate
```

### 2. 依存パッケージのインストール

```bash
python -m pip install --upgrade pip
python -m pip install -r requirements.txt
python -m pip install -r requirements-dev-api.txt
```

### 3. pytest の実行

```bash
FASTAPI_APP="app.main:app" python -m pytest -q
```

## GitHub Actions の設定

`.github/workflows/ci.yml` に以下のワークフローを定義します。

```yaml
name: ci

on:
  push:
  pull_request:
  workflow_dispatch:

jobs:
  api-tests:
    runs-on: ubuntu-latest
    timeout-minutes: 10

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-python@v5
        with:
          python-version: "3.10"
          cache: "pip"

      - name: Install deps (server + api test)
        run: |
          python -m pip install --upgrade pip
          python -m pip install -r backend/requirements.txt
          python -m pip install -r backend/requirements-dev-api.txt

      - name: Run pytest (API)
        env:
          FASTAPI_APP: "app.main:app"
        run: |
          cd backend
          python -m pytest -q --maxfail=1
```


### GitHub Actions

GitHub Actions で green を確認

![GitHub Actions Green](github_actions_green.png)


### ワークフローの設計意図

- 最初は**テストの実行のみ**に集中します（linter や型チェックは後回し）
- `--maxfail=1` オプションで、テストが失敗した際に最短で原因を特定できるようにします
- 外部I/Oを使用しないため、CI環境が安定します

## ハマりやすいポイント

### 1. 作業ディレクトリの問題

`backend/tests/conftest.py` が `from tests...` で import している場合、リポジトリ直下から `pytest` を実行すると `ModuleNotFoundError: No module named 'tests'` というエラーが発生します。

**解決策**: API テストは必ず `cd backend` してから実行することをルールとして徹底します。README と CI の両方で統一しましょう。

### 2. anyio の依存関係

pytest で async テストを実行する際、anyio が必要になります。`anyio>=4.0.0` を `requirements-dev-api.txt` に明示的に記載することで、CI環境でも意図が明確になり、動作が安定します。


## まとめ

この記事では、FastAPI の最小限のエンドポイント（`/health` と `/api/items`）を対象に、pytest + httpx（ASGITransport）を使用してサーバ起動不要のAPIテストを作成し、GitHub Actions でCI環境を構築しました。

今回のPoCでは、API用（pytest）とE2E用（Playwright）で仮想環境を分け、pytest の探索範囲も backend/tests に固定しました。目的は「依存と実行対象の混線」を避け、再現性を保つためです。