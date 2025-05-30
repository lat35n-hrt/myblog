+++
date = '2025-05-10T12:39:41+09:00'
draft = false
title = 'Hugo にカスタムメニューを追加（カテゴリ）'
categories = ["Hugo"]
+++


Hugo でカテゴリーメニューを追加し、多言語サイトに対応させる方法を解説します。日本語をデフォルト言語としつつ、明示的に `/ja/` を URL に含める設計とすることで、Cloudflare Pages 上でも安定したルーティングが可能になります。


![](hugo_menu_categories_ja.png)

---

## 1. 各記事にカテゴリを設定する

記事の Front Matter に `categories` を配列で記載します：

```toml
+++
date = '2025-05-08T16:45:11+09:00'
draft = false
title = 'Hugo にカスタムメニューを追加'
categories = ["Hugo"]
+++

```

---

## 2. hugo.toml に taxonomy を定義

```toml
[taxonomies]
  category = "categories"
```

---

## 3. 自動生成されるカテゴリーページのテンプレート作成

terms.html は taxonomy（分類）一覧ページのテンプレートです。

layouts/_default/terms.html に以下を作成します：


```go
{{ define "main" }}
  <h1>{{ i18n "allCategories" }}</h1>

  {{ range $name, $pages := .Site.Taxonomies.categories }}
    <h2>{{ $name }} ({{ len $pages }})</h2>
    <ul>
      {{ range $pages }}
        <li><a href="{{ .RelPermalink }}">{{ .Title }}</a></li>
      {{ end }}
    </ul>
    <br>
  {{ end }}
{{ end }}

```


---

## 4. カテゴリーページのマークダウン設定

content/ja/categories.md の扱いについて。

以前は「工事中」のテキストを直接書いていましたが、taxonomy ページとしては terms.html が優先されます。

url と layout をFront Matter に明示的に定義しています。


```toml

+++
date = '2025-05-08T10:04:52+09:00'
draft = false
title = 'カテゴリ'
url = "/ja/categories/"
layout = "terms"
+++

```



---

## 5. 多言語対応：i18n による翻訳

共通テンプレートのテキストを日本語化するには、i18n/ ディレクトリを作成し、翻訳キーを定義します。

例：terms.html 内の “All Categories” を翻訳する場合

```text
myblog/
├── i18n/
│   ├── ja.yaml
│   └── en.yaml

```

```yaml
# i18n/ja.yaml
# "All Categories" に対応する日本語表示を定義
allCategories:
  other: "すべてのカテゴリ"
```

```yaml
# i18n/en.yaml
allCategories:
  other: "All Categories"
```

```GoHTML
<h1>{{ i18n "allCategories" }}</h1>

```

---

## 6. URL 設計と defaultContentLanguageInSubdir の設定

・URL設計

カテゴリメニューのリンク先は以下のいずれか：

/ja/categories/

/en/categories/

・defaultContentLanguageInSubdir の設定

デバッグ中に find . -name "index.html" で index.html の存在を確認していたところ、
以下のようにルートにも生成される問題がありました：

./public/categories/index.html

./public/ja/categories/index.html

./public/en/categories/index.html


意図せずルートに categories が生成されるため、hugo.toml に以下の設定を追加：

```toml
defaultContentLanguageInSubdir = true # Include /ja/ in URLs even for the default language to avoid root-level pages
```

ルートに /categories/ が存在すると、Cloudflare Pages などのホスティング環境で意図しないルーティングが発生する可能性があります。



