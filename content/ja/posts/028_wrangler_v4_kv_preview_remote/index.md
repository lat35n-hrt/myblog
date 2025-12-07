+++
date = '2025-12-07T11:52:18+09:00'
draft = false
title = 'Cloudflare Workers + KV Preview/Production セットアップ'
categories = ["cloudflare"]
+++


**個人開発ログ & 再現性確保のためのセットアップガイド**

## 概要

このガイドでは、Cloudflare Workers（Wrangler v4）と Workers KV を使った最小構成の UI プロトタイプの構築方法を解説します。

**目的**: Cloudflare KV から JSON 記事を取得し、最小限の HTML UI にレンダリングする。そして、Cloudflare のツールチェーンが将来変更されても、確実に再現できるように手順を明確に記録すること。


### 開発環境

この記事の動作検証は以下の環境で行いました：

- macOS: 13.7.2 (Intel)
- Node.js: 20.2.0
- npm: 9.6.6
- Wrangler: 4.53.0
- Cloudflare Workers Runtime: 2025-12-05

### このガイドで記録している内容

- 実際に行った操作手順のすべて
- Wrangler v3 と v4 の違い
- Preview KV の生成方法
- dev モードで remote KV を有効にする方法
- UI が Preview KV を正しく読み取れるようになるまでの流れ

かなり冗長に書いていますが、個人の開発ログとしての役割も兼ねています。

---

## 1. プロジェクト初期化

```bash
mkdir your-app
cd your-app
wrangler init .
npm install --save-dev wrangler
```

Wrangler により以下が生成されます:

- `src/index.ts`
- `wrangler.jsonc`（後で `wrangler.toml` に置き換える）
- TypeScript の設定ファイル / テストファイル

---

## 2. wrangler.jsonc → wrangler.toml への置き換え

JSONC を TOML に置き換えました。
理由1: 別の作業で使っていた wrangler.toml に統一したかった。
理由2: トラブルシューティングで視認性が高い。

```bash
rm wrangler.jsonc
touch wrangler.toml
```

初期内容:

```toml
name = "your-app"
main = "src/index.ts"
compatibility_date = "2025-12-05"

[observability]
enabled = true
```

---

## 3. Workers KV ネームスペースの作成（Production + Preview）

Cloudflare KV では以下が必要です:

- **id** — 本番用 Namespace
- **preview_id** — 開発・プレビュー用 Namespace

**重要**: Wrangler v4 は（v3 と異なり）`preview_id` を自動生成しません。そのため Production / Preview の両方を手動で作成します。

### 3.1 Production KV Namespace

```bash
npx wrangler kv namespace create test_kv
```

返ってきた値:

```
id = "abcde12345"
```

### 3.2 Preview KV Namespace（手動作成 — Wrangler v4 必須）

v4 は preview KV を自動作成しないため、明示的に生成します。

```bash
npx wrangler kv namespace create test_kv --preview
```

返ってきた値:

```
preview_id = "fghi67890"
```

> ⚠️ **注意**: 大半のオンライン記事でこの記述が抜けています。Wrangler v4 の dev 動作が変更されたためです。

---

## 4. wrangler.toml の KV 設定

最終的な設定:

```toml
[[kv_namespaces]]
binding = "test_kv"
id = "abcde12345"        # Production KV
preview_id = "fghi67890" # Dev KV (Preview)
remote = true            # dev モードで remote KV を強制
```

### `remote = true` が必要な理由

- Wrangler v4 のデフォルトは **local Miniflare KV**
- `preview_id` を指定しても local KV が優先される
- そのため Worker は常に `RAW: null` を返していた
- **`remote = true` を指定した瞬間、dev モードは Preview KV を正しく読み始めました**

これは v3 → v4 の大きな仕様変更点の一つです。

---

## 5. KV にサンプルデータを投入する

### ローカル KV への書き込み

```bash
npx wrangler kv key put latest_articles "$(cat ./data/latest_articles.json)" --binding=test_kv --local
```

### Preview（remote KV）への書き込み

```bash
npx wrangler kv key put latest_articles \
  "$(cat ./data/latest_articles.json)" \
  --binding=test_kv --remote
```

### 検証

```bash
npx wrangler kv key get latest_articles --binding=test_kv --remote
```

正しい結果:

```json
[
  {
    "title": "Example Article",
    "url": "https://example.com",
    "summary": "This is a test summary."
  }
]
```

---

## 6. Worker コード（src/index.ts）

raw KV 出力をログに表示し、簡易 HTML を返す最小構成の Worker:

```typescript
export default {
  async fetch(request, env) {
    const raw = await env.test_kv.get("latest_articles");
    console.log("RAW:", raw);

    let articles = [];
    try {
      articles = raw ? JSON.parse(raw) : [];
    } catch {}

    const html = `
      <html>
      <body>
        <h1>📰 NewsLite UI (PoC)</h1>
        ${articles.map(a => `
          <div>
            <a href="${a.url}">${a.title}</a>
            <p>${a.summary}</p>
          </div>
        `).join("")}
      </body>
      </html>
    `;

    return new Response(html, {
      headers: { "Content-Type": "text/html; charset=UTF-8" },
    });
  },
};
```

---

## 7. Worker の実行（dev モード）

```bash
npx wrangler dev
```

期待されるログ:

```
Your Worker has access to:
env.test_kv  KV Namespace   remote
```

`local` → `remote` に変わった瞬間、ブラウザに KV データが表示され始めます。

---

## 8. 動作結果

以下にアクセス:

```
http://localhost:8787/
```

正しく表示:

```
📰 NewsLite UI (PoC)

Example Article
This is a test summary.
```

これにより次が確認できました:

- ✅ Preview KV が正しく動作
- ✅ Worker コードは正しい
- ✅ wrangler.toml の設定も正しい
- ✅ remote binding が正常に機能

---

## 9. v3 → v4：重要な動作仕様の違い

デバッグが難しくなった原因となる概念的な違いです。

### 9.1. KV Preview の挙動

| Wrangler version | 動作 |
|------------------|------|
| v3.x | `preview_id` → `wrangler dev` で自動的に使用 |
| v4.x | `preview_id` があっても local Miniflare KV が使用される |

### 9.2. Remote KV アクセス

| Action | v3 の挙動 | v4 の挙動 |
|--------|-----------|-----------|
| dev モードでの KV 読み取り | Remote Preview KV | `remote = true` なし → Local KV |
| preview_id 必須？ | 任意 | remote dev では必須 |
| remote フラグは必要？ | 不要 | 必要（`remote = true`） |

### 9.3. その結果どうなるか

Wrangler v4 では、Cloudflare Dashboard に正しい KV データがあっても、Worker は local KV を読んでしまうため `RAW: null` を返す。

まさに今回デバッグした問題そのものです。

---

## 10. まとめ

### 重要なポイント

- ⚠️ **Wrangler 4 はドキュメントに明記されていない破壊的変更を含む**
- 📚 多くのチュートリアルがまだ Wrangler 3 の挙動で書かれている
- 🔧 **Preview KV は手動生成が必須**
- 🌐 **`remote = true` がなければ dev モードで remote KV は読めない**

このガイドが、Cloudflare Workers + KV を使った開発の再現性確保に役立てば幸いです。