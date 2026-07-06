+++
date = '2026-07-06T14:42:59+09:00'
draft = false
title = 'GitHub ActionsでPlaywright E2Eテストを実行する（React + FastAPI + SQLite）'
categories = ["playwright"]
+++


これまでの記事では、Playwrightの基本的な使い方や、GitHub Actionsを利用したCIの導入について紹介しました。

今回は、個人開発中の EventLens プロジェクトを対象に、GitHub Actions上でPlaywrightによるE2Eテストを自動実行する仕組みを構築しました。

対象となる技術スタックは以下の通りです。

* React (Vite)
* FastAPI
* SQLite
* Playwright
* GitHub Actions

目標は、GitHubへコードをPushした際にE2Eテストを自動実行し、アプリケーション全体の動作を継続的に検証できるようにすることです。

---

# プロジェクト構成

```text id="42nrr6"
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

フロントエンドはReact + Viteで構築されており、FastAPIバックエンドと通信しています。

---

# CIワークフローの全体像

今回のGitHub Actionsでは、以下の順序で処理を実行します。

1. ソースコードの取得
2. Node.js環境の準備
3. フロントエンド依存関係のインストール
4. Python環境の準備
5. バックエンド依存関係のインストール
6. SQLiteデータベースの初期化
7. FastAPIの起動
8. Playwrightブラウザのインストール
9. Playwrightテストの実行
10. HTMLレポートの保存

概念図にすると次のようになります。

```text id="ax2f7g"
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

# Node.js環境の準備

Node.jsのバージョンはローカル開発環境と揃えるため、`.nvmrc` を利用しています。

```yaml id="e7um9s"
- name: Setup Node.js
  uses: actions/setup-node@v4
  with:
    node-version-file: frontend/.nvmrc
```

これにより、ローカル環境とGitHub Actions環境で同じNode.jsバージョンを利用できます。

---

# Python環境の準備

FastAPIを起動するため、Pythonもセットアップします。

```yaml id="yb96zw"
- name: Set up Python
  uses: actions/setup-python@v5
  with:
    python-version: "3.10"

- name: Install Python dependencies
  run: pip install -r requirements.txt
```

バックエンド依存関係は `requirements.txt` からインストールします。

---

# SQLiteデータベースの初期化

SQLiteのDBファイル自体はGit管理していません。

代わりに、CI実行時に毎回生成する方式を採用しています。

```yaml id="eyg0cg"
- name: Initialize SQLite database
  run: |
    sqlite3 data/eventlens.db < app/db/schema.sql
    sqlite3 data/eventlens.db < data/seeds/seed.sql
```

この方法により、毎回クリーンな状態からテストを実行できます。

データベースファイルをコミットする必要もありません。

---

# FastAPIの起動

PlaywrightのE2EテストではバックエンドAPIが必要になるため、FastAPIを起動します。

```yaml id="wlpgoa"
- name: Start FastAPI server
  run: |
    uvicorn app.main:app --host 127.0.0.1 --port 8001 &
```

コマンドの末尾に付けている `&` は、FastAPI をバックグラウンドプロセスとして起動するためのものです。

もし `&` を付けない場合、GitHub Actions は FastAPI プロセスの終了を待ち続けるため、後続の Playwright テストが実行されません。

バックグラウンドで起動することで、FastAPI を動かしたまま次の Step に進み、Playwright から API を利用した E2E テストを実行できます。

---

# Playwrightブラウザのインストール

今回の構築で最も理解が深まったポイントがここです。

私は当初、

```text id="u7v8dc"
npm ci
```

だけでPlaywrightのインストールが完了すると考えていました。

しかし実際には、

```text id="ikdf8j"
Playwrightライブラリ

と

ブラウザ実体
```

は別管理になっています。

まず、

```text id="nn1t0i"
npm ci
```

でインストールされるのは、

```text id="cnqy4d"
@playwright/test
```

です。

一方で、ChromiumやFirefoxなどのブラウザ本体は別途インストールが必要です。

```yaml id="vwhu2w"
- name: Install Playwright browsers
  working-directory: frontend
  run: npx playwright install --with-deps
```

CI/CD環境でPlaywrightを利用する際は、この違いを理解しておくことが重要です。

---

# Playwrightテストの実行

準備が整ったら、E2Eテストを実行します。

```yaml id="s59iq3"
- name: Run Playwright tests
  working-directory: frontend
  run: npm run test:e2e
```

今回のEventLensプロジェクトでは、以下の4件のテストが正常に実行されました。

```text id="jv9s6r"
Running 4 tests using 2 workers

✓ loads and shows seeded events
✓ filters CME only
✓ search keyword Oncology
✓ pagination

4 passed
```

---

# Playwright HTMLレポートの保存

テスト結果はGitHub ActionsのArtifactとして保存します。

```yaml id="jlwmg7"
- name: Upload Playwright report
  if: always()
  uses: actions/upload-artifact@v4
  with:
    name: playwright-report
    path: frontend/playwright-report/
    retention-days: 14
```

ポイントは以下の2点です。

* `if: always()` によりテスト失敗時もレポートを保存
* Artifactは14日間保存

ArtifactはGitリポジトリには保存されません。

GitHub Actions実行結果画面からダウンロードできます。

---

# 学んだこと

今回の構築を通じて、PlaywrightをCIで動かすためには複数のレイヤーが存在することを理解できました。

```text id="6qf6pq"
Node.js

↓

npm依存関係

↓

Playwrightライブラリ

↓

ブラウザ実体

↓

FastAPI

↓

SQLite

↓

E2Eテスト
```

特に、

「Playwrightをインストールする」

という言葉の中に、

* npmパッケージのインストール
* ブラウザ実体のインストール

という2種類の処理が含まれている点は大きな学びでした。

---

# まとめ

今回、React + FastAPI + SQLite構成のアプリケーションに対して、GitHub Actions上でPlaywright E2Eテストを自動実行するCIパイプラインを構築しました。

最終的に実現できたことは以下の通りです。

* GitHub Pushで自動実行
* SQLiteデータベースの自動生成
* FastAPIの自動起動
* Playwrightブラウザの自動セットアップ
* E2Eテストの自動実行
* HTMLレポートのArtifact保存

これにより、ローカル環境だけでなく、クリーンなGitHub Runner環境でもアプリケーションが正常に動作することを継続的に検証できるようになりました。

---

# 次回予告

今回の構成では、GitHub Actions実行時に毎回Playwrightブラウザをインストールしています。

次回は Playwright公式Dockerイメージを利用し、

* ブラウザインストールの省略
* ワークフローの簡素化
* Docker利用時との比較

について検証してみたいと思います。
