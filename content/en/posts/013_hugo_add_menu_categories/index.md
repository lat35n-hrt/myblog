+++
date = '2025-05-10T12:39:48+09:00'
draft = false
title = 'Adding a Custom Menu in Hugo (Category)'
categories = ["Hugo"]
+++

This article explains how to add a category menu in Hugo and support multilingual sites. By explicitly including `/ja/` in the URL even for the default language (Japanese), you can ensure stable routing on platforms like Cloudflare Pages.

---

## 1. Assign Categories to Each Post

Add the `categories` field as an array in each post’s Front Matter:

```toml
+++
date = '2025-05-08T16:45:11+09:00'
draft = false
title = 'Adding a Custom Menu in Hugo'
categories = ["Hugo"]
+++

```
---

## 2. Define Taxonomies in hugo.toml

```toml
[taxonomies]
  category = "categories"
```

---

## 3. Create Template for Auto-Generated Category Page

terms.html is the template for taxonomy listing pages.
Create the following file at layouts/_default/terms.html:

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

## 4. Markdown Setup for Category Page
Regarding content/ja/categories.md:
Previously, you might have written “Under Construction” directly in this file. However, once it functions as a taxonomy page, terms.html will take precedence.

Explicitly define url and layout in its Front Matter:


```toml

+++
date = '2025-05-08T10:04:52+09:00'
draft = false
title = 'Categories'
url = "/ja/categories/"
layout = "terms"
+++

```


---

## 5. Multilingual Support: Using i18n for Translations
To localize text in shared templates, create an i18n/ directory and define translation keys.
Example: Translating “All Categories” in terms.html

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

## 6. URL Structure and defaultContentLanguageInSubdir Setting
URL Design
The category menu should point to either of the following:

/ja/categories/

/en/categories/

defaultContentLanguageInSubdir Setting
While debugging using find . -name "index.html", you may find:


./public/categories/index.html

./public/ja/categories/index.html

./public/en/categories/index.html


To avoid /categories/ being unexpectedly generated at the root level, add this setting to hugo.toml:

```toml
defaultContentLanguageInSubdir = true # Force /ja/ in URLs even for default language
```

Having /categories/ at the root can cause unintended routing behavior on hosting platforms like Cloudflare Pages.




