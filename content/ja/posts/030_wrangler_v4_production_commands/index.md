+++
date = '2025-12-17T17:43:23+09:00'
draft = false
title = 'Wrangler v4 本番運用コマンド整理（Production 編）'
categories = ["cloudflare"]
+++


本記事では、Cloudflare Workers + KV / R2 を使った PoC で使ったコマンドと考え方を整理します。

開発（wrangler dev）ではなく、**Production KV / R2 を意識した運用フェーズ**が対象です。

[前回の記事](https://tarclog.com/ja/posts/028_wrangler_v4_kv_preview_remote/)では preview までだったので、Production 編では運用で動作したコマンドの整理となります。

## 前提整理（重要）

wrangler v4 では、以下の軸を意識しないと混乱します。

| 観点 | 値 |
|------|-----|
| Worker 実行 | ローカル or Cloudflare |
| KV / R2 | local（エミュレーション） or remote（実体） |
| Namespace | preview or production |

本記事では **Production KV / R2 を「確実に触る」** ことを優先します。

## 1. 本番 KV への JSON PUT（production）

### 推奨コマンド（確定版）
```bash
npx wrangler kv key put articles/2025-12-13 \
  "$(cat data/daily_summary_2025-12-13_with_audio.json)" \
  --binding=newslite_kv \
  --remote \
  --preview=false
```

### ポイント

- `--remote`: Cloudflare 上の KV に必ず書き込む
- `--preview=false`: production namespace（id）を明示的に指定
- `cat` 方式を採用
  - 実体をその場で確認できる
  - CI / shell script にも組み込みやすい

## 2. 本番 KV の GET（確認用）
```bash
npx wrangler kv key get articles/2025-12-13 \
  --binding=newslite_kv \
  --remote \
  --preview=false
```

### 目的

- 書き込み成功の即時確認
- 「どの namespace を見ているか」を明確にする

## 3. wrangler dev（本番データ確認用）

### 基本形
```bash
npx wrangler dev
```

### 重要な補足（v4 の挙動）

- `wrangler dev` は Worker Runtime をローカルで起動
- ただし **KV / R2 は設定次第で remote を参照可能**

`wrangler.toml` に以下がある場合：
```toml
[[kv_namespaces]]
binding = "newslite_kv"
id = "..."
preview_id = "..."
remote = true
```

➡ localhost でも production / preview KV を直接読む

### 実際の確認 URL
```
http://localhost:8787/?date=2025-12-13
```

## 4. wrangler deploy（Worker 本体の反映）
```bash
npx wrangler deploy
```

### いつ deploy が必要か?

| ケース | deploy 必要？ |
|--------|--------------|
| KV / R2 の更新のみ | ❌ 不要 |
| index.ts の変更 | ✅ 必須 |
| リダイレクト仕様変更 | ✅ 必須 |
| HTML / UI 修正 | ✅ 必須 |

**KV / R2 は Worker と独立しているため、「データ更新だけなら deploy は不要」** というのは v4 でも変わりません。

## 5. 本番 URL での動作確認

### workers.dev（Cloudflare 提供）
```
https://<worker-name>.<your-subdomain>.workers.dev/?date=2025-12-13
```

### Custom Domain（本番想定）
```
https://newslite.tarclog.com/?date=2025-12-13
```

※ Custom Domain を設定しても workers.dev は引き続き利用可能
（R2.dev とは別概念）

## 6. よくある混乱ポイント（v4 特有）

### ❌ `wrangler dev --remote`

v4 では廃止。`remote = true` を toml 側で制御する設計

### ❌ `key:put` / `key:get`

正式には `key put` / `key get`
コロンは不要（GPT が誤りがち）

## 7. 運用フロー（NewsLite 実例）

1. JSON生成（ローカル）
2. audio URL 自動付与
3. R2 に mp3 アップロード
4. KV (preview) put → localhost 確認
5. KV (production) put
6. workers.dev / custom domain 確認
7. index.ts 変更があれば deploy

## まとめ

- wrangler v4 は「ローカル志向」だが、remote 強制は可能
- 本番運用では `--remote` + `--preview=false` を必ず明示
- KV / R2 更新と Worker deploy は別物
- 「どの namespace を触っているか」を常に意識する