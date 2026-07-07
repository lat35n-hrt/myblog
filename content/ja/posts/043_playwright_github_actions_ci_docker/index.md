+++
date = '2026-07-07T15:03:21+09:00'
draft = false
title = 'Playwright + GitHub Actions + Docker で E2E テストを構築'
categories = ["playwright"]
+++


## はじめに

前回の記事では、Playwright と GitHub Actions を利用して React アプリケーションの E2E テストを自動化しました。

今回はさらに一歩進め、**Playwright 公式 Docker イメージ**を利用した CI を構築しました。

目的は Docker 化することではなく、

- GitHub Runner を利用する通常版との違い
- Docker イメージに最初から含まれるもの
- Docker でも追加設定が必要なもの

を実際に検証することです。

---

## 開発環境

- Frontend: React + Vite
- Node.js: 22.12.0
- Backend: FastAPI
- Python: 3.10
- Database: SQLite
- Test Framework: Playwright 1.57.0
- CI: GitHub Actions
- Docker：Playwright Official Image

Docker イメージには以下を使用しました。

```text
mcr.microsoft.com/playwright:v1.57.0-noble
```

Playwright のバージョンと Docker イメージのバージョンを合わせています。

---

# なぜ Docker 版を試したのか

通常版では GitHub Runner 上で次のような準備を行っていました。

- Node.js のセットアップ
- Playwright ブラウザのインストール
- Python のセットアップ
- SQLite データベース作成
- FastAPI 起動
- Playwright 実行

一方、Playwright 公式 Docker イメージには

- Node.js
- Chromium
- Firefox
- WebKit
- Playwright が必要とするブラウザ依存ライブラリ

があらかじめ含まれています。

つまり、

```yaml
npx playwright install --with-deps
```

は不要になることが期待できます。

---

# Docker Workflow

今回は通常版とは別に

```text
.github/workflows/playwright-docker.yml
```

を作成しました。

実行は

```yaml
on:
  workflow_dispatch:
```

として、手動実行にしています。

通常版の CI を壊さずに Docker 版だけを検証できるためです。

---

# 実装中に遭遇した問題

今回は完成までにいくつかのエラーが発生しました。

一つずつ原因を調査しながら修正していきました。

---

## ① container の記述位置

最初は

```yaml
container:
```

を Workflow のトップレベルに記述していました。

しかし GitHub Actions では

```yaml
jobs:
  e2e:
    container:
```

のように **Job の中**へ記述する必要があります。

これを修正すると Workflow が正常に開始しました。

---

## ② setup-node は不要だった

通常版では

```yaml
uses: actions/setup-node@v4
```

を使用していました。

しかし Docker イメージには Node.js が含まれているため、

この Step を削除しても正常に動作しました。

Docker のメリットを一つ確認できました。

---

## ③ pip が見つからない

Python セットアップ後に

```bash
pip install -r requirements.txt
```

を実行すると

```text
pip: not found
```

というエラーになりました。

そこで

```bash
python -m pip install -r requirements.txt
```

へ変更したところ正常に動作しました。

CI 環境ではこちらの書き方の方が安定しています。

---

## ④ SQLite が存在しない

次に発生したエラーは

```text
sqlite3: not found
```

でした。

Playwright Docker イメージには SQLite CLI は含まれていません。

そのため

```bash
apt-get update
apt-get install -y sqlite3
```

を追加して解決しました。

Playwright 用コンテナであり、アプリケーション開発用コンテナではないことが分かります。

---

## ⑤ Playwright のバージョン

最後に

```text
Executable doesn't exist...
Please update docker image as well.
```

というエラーが発生しました。

原因は

- package.json
- Docker イメージ

の Playwright バージョンが一致していなかったためです。

Docker イメージを

```text
mcr.microsoft.com/playwright:v1.57.0-noble
```

へ変更することで解決しました。

Docker 利用時は Playwright のバージョンを合わせることが重要です。

---

# 通常版との比較

| 通常版 | Docker版 |
|--------|----------|
| setup-node | 不要 |
| Playwright Browser Install | 不要 |
| setup-python | 必要 |
| python -m pip | 推奨 |
| SQLite CLI | 追加インストール |
| FastAPI | 必要 |
| Playwright Test | 同じ |

Docker 化してもアプリケーション側のセットアップは必要ですが、ブラウザ環境の準備が不要になる点は大きなメリットでした。

---

# 完成した CI

最終的に Docker Workflow は

- Node.js（Docker イメージ）
- npm install
- Python セットアップ
- Python ライブラリインストール
- SQLite 初期化
- FastAPI 起動
- Playwright 実行
- HTML レポートを Artifact 保存

まで自動化できました。

通常版と Docker 版の両方を比較できる構成になっています。

---

# 学んだこと

今回の検証で最も印象に残ったのは、

**「Docker イメージには何でも入っているわけではない」**

という点でした。

Playwright に必要なものは最初から含まれていますが、

FastAPI や SQLite などアプリケーション固有の環境は自分で構築する必要があります。

また、

- GitHub Actions
- Docker
- Node.js
- Python
- SQLite
- Playwright

それぞれがどの役割を担っているかを実際に検証しながら理解することができました。