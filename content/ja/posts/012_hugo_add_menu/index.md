+++
date = '2025-05-08T16:45:11+09:00'
draft = false
title = 'Hugo にカスタムメニューを追加'
categories = ["Hugo"]
+++


## 環境

- Hugo v0.146
- macOS(13.7.2) → GitHub → Cloudflare Pages

## 目的

Hugo で構築したブログに、日本語・英語それぞれのメニュー（「このブログについて」(About)と「カテゴリ」（Categories）を追加します。

## 設定方法

`hugo.toml` の `languages` セクションに、以下のように各言語を入れ子にして設定します。

````toml
[languages]
  [languages.ja]
    languageName = "日本語"
    weight = 1
    contentDir = "content/ja"
    title = "TarcLog｜日々の学びと発信"
    languageCode = "ja"
    params.homeInfoParams = { Title = "ようこそ", Content = "ここでは日々の学びや試行錯誤を共有しています。" }

    [[languages.ja.menu.main]]
      identifier = "about"
      name = "このブログについて"
      url = "/ja/about/"
      weight = 10

    [[languages.ja.menu.main]]
      identifier = "categories"
      name = "カテゴリ"
      url = "/ja/categories/"
      weight = 20

  [languages.en]
    languageName = "English"
    weight = 2
    contentDir = "content/en"
    title = "TarcLog | Learning and Sharing Every Day"
    languageCode = "en"
    params.homeInfoParams = { Title = "Welcome!", Content = "This is where I share my daily learning and trial-and-error experiences." }

    [[languages.en.menu.main]]
      identifier = "about"
      name = "About"
      url = "/en/about/"
      weight = 10

    [[languages.en.menu.main]]
      identifier = "categories"
      name = "Categories"
      url = "/en/categories/"
      weight = 20


````


# トラブルシューティング
「カテゴリ」の日本語個別記事をカテゴリメニューから表示した際に、ページ内容は日本語なのにタイトルが英語のままになるという問題が発生しました。

対処方法
各記事の front matter に url を明示的に設定することで、正しく表示されるようになります。

````toml
+++
date = '2025-05-07T21:11:04+09:00'
draft = false
title = 'このブログについて'
url = "/ja/about/"
+++

````

これにより、Hugo が自動で生成する URL よりも url の指定が優先され、言語別ルーティングが正しく機能します。
