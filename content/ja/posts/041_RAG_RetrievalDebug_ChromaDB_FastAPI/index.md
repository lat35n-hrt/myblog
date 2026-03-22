+++
date = '2026-03-22T15:24:56+09:00'
draft = false
title = 'FastAPI + ChromaDB による RAG Retrieval デバッグ PoC'
categories = ["AI"]
tags = ["RAG", "Retrieval", "ChromaDB", "FastAPI", "Debugging"]
+++


# RAGで「古い情報が上位に来る問題」をどう潰すか


---

## 背景

RAG（Retrieval-Augmented Generation）を使ったナレッジボットでは、しばしば次の問題に遭遇します。

> **最新ドキュメントではなく、古い情報が上位にヒットする**

これは単なるデータ不備ではなく、**検索（retrieval）の設計問題**です。
今回、FastAPI + ChromaDB を使った最小構成のPoCで、この問題を**再現 → 原因特定 → 最小修正**まで行いました。

---

## 再現した問題

次のような2つのドキュメントを用意します：

* **v4（最新）**：公式ドキュメント（正しいが表現が抽象的）
* **v3（旧）**：ブログ記事（古いがクエリと強く一致する表現）

クエリ：

```
Which guidance should I prefer when wrangler v3 and v4 instructions conflict?
```

### 結果（問題ケース）

* v3（古い）が上位ヒット
* v4（正しい）が下位に落ちる

理由はシンプルです：

> **embedding 類似度は「意味の近さ」であり「正しさ」ではない**

---

## 原因の分解

この問題は主に3つの要因で発生します。

### 1. 類似度偏重（semantic similarity bias）

クエリに近い文言を持つ古いドキュメントが優先される

### 2. スコープ制御の欠如

検索対象が「全データ」になっている

### 3. データ汚染（重複・旧データ残存）

同一ドキュメントの旧バージョンが残り続ける

---

## アプローチ：最小構成での修正

このPoCでは「大きな設計変更」はせず、**ピンポイント修正**に限定しました。

---

### ① Metadataフィルタで検索スコープを制御

ChromaDB の `where` を利用して検索対象を制限します。

```json
{
  "source_type": "official_docs",
  "version_major": 4
}
```

内部的には `$and` 条件として適用：

```python
where = {
  "$and": [
    {"source_type": "official_docs"},
    {"version_major": 4}
  ]
}
```

#### 効果

* 古い blog (v3) を完全に排除
* 正しいドキュメントのみ検索対象に

---

### ② Idempotent ingest（重複防止）

```python
# ingest前に同一doc_idを削除
collection.delete(where={"doc_id": doc_id})
```

#### 効果

* 同じドキュメントの多重登録を防止
* 古いchunkが残る問題を回避

---

### ③ Debug endpoint で可視化

```
POST /debug/retrieval
```

取得情報：

* documents
* metadata
* distance（類似度）

#### 例（出力イメージ）

```json
{
  "results": [
    {
      "doc_id": "wrangler_v4",
      "distance": 0.21,
      "metadata": {
        "source_type": "official_docs",
        "version_major": 4
      }
    }
  ],
  "where": {
    "$and": [...]
  }
}
```

#### 効果

* 「なぜそれが選ばれたか」が説明可能になる
* ブラックボックス化を回避

---

## 修正前 / 修正後

| 状態              | 結果         |
| --------------- | ---------- |
| Before          | v3（古い）が上位  |
| After（filter適用） | v4（最新）のみ取得 |

---

## 設計上のポイント

### 1. Retrieval は「検索」ではなく「制御」

単に類似度に任せると破綻します。

* 類似度 → ranking
* metadata → scope control

この2つを分離するのが重要です。

---

### 2. DBでできない条件はアプリで補う

Chromaの制約：

* `published_at >=` のような比較が弱いケースあり

対策：

```python
# Python側で後処理フィルタ
if doc["published_at"] >= min_published_at:
    filtered.append(doc)
```

---

### 3. Debug機構は最初から入れる

後付けすると調査コストが跳ねます。

今回のPoCでは：

* dev専用 (`DEBUG_MODE=1`)
* 本番では無効化

---

## このPoCの価値

この構成は、実務でよくある次の依頼に直結します：

* 「古い情報を出さないようにしたい」
* 「検索結果の理由を説明したい」
* 「RAGの精度が不安定」

そして重要なのは：

> **大規模な再設計なしで改善できる**

---

## まとめ

今回のポイントを一行でまとめると：

> **RAGの品質問題は、検索範囲の制御とデータ衛生で大半が解決する**

具体的には：

* metadata filter（where）
* idempotent ingest
* debug endpoint

この3点で「古い情報が上位に来る問題」は再現・説明・修正まで一貫して扱えます。

---

## 次のステップ（発展）

* re-ranking（cross-encoder）
* hybrid search（BM25 + embedding）
* time decay（新しいほどスコア加点）

ただし、今回のように：

> **まずは単純なフィルタで直るか？**

を確認するのが最短ルートです。

---

## Repo

PoCコードは[こちら](https://github.com/lat35n-hrt/rag_retrieval_diagnostics_poc)：


* FastAPI（単一ファイル）
* ChromaDB（ローカル永続化）
* curlで再現可能な検証フロー

---

必要十分な最小構成で、RAGの「よくある失敗」を潰すための実装例として参考になれば幸いです。
