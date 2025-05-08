+++
date = '2025-05-08T16:45:18+09:00'
draft = false
title = 'Adding a Custom Menu in Hugo'
+++


## Environment

- Hugo v0.146
- macOS(13.7.2) → GitHub → Cloudflare Pages

## Objective

I wanted to add language-specific menus (About and Categories) to my Hugo-based blog, supporting both Japanese and English.

## Configuration

In `hugo.toml`, define the `languages` section with nested entries for each language, like this:

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


## Troubleshooting
When accessing the Japanese "Categories" page via the menu, the content correctly appears in Japanese, but the title was displayed in English.

Solution
To fix this, I explicitly added the url to the front matter of the corresponding content file:


````toml
+++
date = '2025-05-07T21:11:04+09:00'
draft = false
title = 'このブログについて'
url = "/ja/about/"
+++

````

This forces Hugo to use the specified path and ensures proper language-based routing.

