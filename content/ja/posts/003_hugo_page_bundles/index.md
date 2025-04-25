+++
date = '2025-04-25T09:38:52+09:00'
draft = false
title = 'Hugoブログ：Page Bundlesの使い方'
+++

# Hugo の Page Bundles について
[公式ドキュメント](https://gohugo.io/content-management/page-bundles/)

---

## 📦 Page Bundles とは？

Hugo では、Markdown ファイルとその関連ファイル（画像やデータなど）を1つのディレクトリにまとめて管理することができます。これを **Page Bundle** と呼びます。

---

## ✅ Leaf Bundle の構成（記事単位）

各記事をディレクトリ単位で管理し、Markdown ファイルは `index.md` に記述します。画像ファイルなども同じディレクトリ内に格納できます。

```text
content/
├── ja/
│   └── posts/
│       └── 002_cloudflare_deploy/
│           ├── index.md
│           └── image.png（任意）
├── en/
│   └── posts/
│       └── 002_cloudflare_deploy/
│           ├── index.md
│           └── image.png（任意）

```

---

## 🖼️ 画像の挿入方法
同じディレクトリ内の画像は、以下のように簡単に読み込めます。

```text
![](image.png)
```

---

## 🌿 Branch Bundle の使いどころ
_index.md を使うことで、そのディレクトリ配下の一覧ページ（セクションページ）をカスタマイズできます。
たとえば、記事一覧のタイトルや説明、目次を表示したい場合などに使用します。

```text
content/
└── ja/
    └── posts/
        ├── _index.md（一覧ページのカスタム）
        └── 001_xxx/
        └── 002_xxx/

```