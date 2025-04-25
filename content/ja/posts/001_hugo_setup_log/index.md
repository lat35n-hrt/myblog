+++
title = "Hugo + PaperMod でブログを立ち上げた記録"
date = 2025-04-23T14:00:00+09:00
draft = false
+++

## 概要

Hugo(v0.146) を使って、静的ブログを立ち上げた記録。

テーマは PaperMod。
日本語と英語の２言語設定。

個人の学習ログ・作業記録として、今後も継続していく予定。



---
## 手順

### 1. 初期セットアップ

```bash
brew install hugo
hugo new site myblog
cd myblog
git init
```

### 2. テーマ導入（PaperMod）

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
```

### 3. `hugo.toml` の設定：

```toml
baseURL = "https://yourdomain.com/"
defaultContentLanguage = "ja"
theme = "PaperMod"

[pagination]
  paginate = 5

[languages]
  [languages.ja]
    languageName = "日本語"
    weight = 1
    contentDir = "content/ja"
    title = "My Blog"
    languageCode = "ja"
    params.homeInfoParams = { Title = "ようこそ", Content = "これは日本語のブログです。" }

  [languages.en]
    languageName = "English"
    weight = 2
    contentDir = "content/en"
    title = "My Blog"
    languageCode = "en"
    params.homeInfoParams = { Title = "Welcome", Content = "This is an English blog." }

[params]
  ShowReadingTime = true
  ShowPostNavLinks = true
```

### 4. 記事作成

```bash
hugo new content/ja/posts/hello-ja.md
hugo new content/en/posts/hello-en.md
```

:::tip
`hugo new` は `contentDir` に依存して意図しない場所にファイルを作ることがあるため、**フルパス指定が確実**。
:::

---
### 5. ローカルサーバを実行

```bash
hugo server
```

ローカル環境でサイトを確認: [http://localhost:1313](http://localhost:1313)

---
### 6. GitHub との連携


- .gitignore に除外項目を追記:


```
public/
.hugo_build.lock
.DS_Store
.vscode/
```

- GitHubに新規レポジトリ `myblog` を作成する
- リモート更新する:

```bash
git add .
git commit -m "initial commit"
git remote add origin https://github.com/ユーザー名/myblog.git
git push -u origin main
```


---

### 7. Cloudflare Pages にデプロイ

- GitHub と連携し、Cloudflare Pages の「+Add > Pages」からデプロイ
- `public/` 配下の静的ファイルが公開される
- Hugo のバージョン(v0.146)とビルドコマンド（`hugo`）を指定
  Build command: hugo
  Build output directory: /public
  Variable name: HUGO_VERSION = 0.146

- hugo.toml の baseURL は仮置きの状態だったので、Cloudflare Pages の URL を設定する。
（トップ画面は表示されるがリンクが機能しないので修正）
---

### 8. カスタムドメインの検討

日本語 + `log` のネーミングを候補に。

---


### 感想・今後の運用

- PaperMod の落ち着いたレイアウトは、文字中心の記録に向いている
- 設定ファイルや長い手順は Notion に、Hugo 側では日々の進捗を記録
- 将来的に英語記事と日本語記事の切り替え運用も視野に

---

この記録は、未来の自分へのメモです。忘れてしまっても、ここを見ればまた立ち上げられるようにしておきます。
