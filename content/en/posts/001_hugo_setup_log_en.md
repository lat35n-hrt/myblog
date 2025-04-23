+++
title = "Hugo Setup Log"
date = 2025-04-23T14:00:00+09:00
draft = false
+++

## Overview

This is a personal setup log for building a multilingual blog using Hugo(v0.146) and the PaperMod theme. The goal was to create a lightweight, text-focused site, mainly as a log for future reference.

---

## Steps

### 1. Hugo Installation and Project Initialization

```bash
brew install hugo
hugo new site myblog
cd myblog
git init
```

### 2. Add PaperMod Theme

```bash
git submodule add https://github.com/adityatelange/hugo-PaperMod.git themes/PaperMod
echo 'theme = "PaperMod"' >> hugo.toml
```

### 3. Set Up Basic `hugo.toml`

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

### 4. Create Posts

Use full path when creating content for multilingual setup:

```bash
hugo new content/ja/posts/hello-ja.md
hugo new content/en/posts/hello-en.md
```

### 5. Run Local Server

```bash
hugo server
```

Check the site locally at: [http://localhost:1313](http://localhost:1313)

---

### 6. GitHub Setup

- Added `.gitignore` to exclude the following:

```
public/
.hugo_build.lock
.DS_Store
.vscode/
```

- Created a new repository `myblog` on GitHub
- Linked remotely:

```bash
git add .
git commit -m "initial commit"
git remote add origin https://github.com/ユーザー名/myblog.git
git push -u origin main
```

---

### 7. Deploy to Cloudflare Pages

- Linked with GitHub and deployed from Cloudflare Pages using “+Add > Pages”
- Static files under the public/ directory are served
- You can also specify the Hugo version(v0.146) and build command ('hugo')

- The baseURL in hugo.toml was a placeholder, so I updated it to the actual Cloudflare Pages URL.
(The homepage was visible, but internal links didn’t work until this was fixed.)
---

### 8. Domain and Naming

Considering domain names with Japanese + log:

---

### 9. Final Notes

The technical logs and configurations (e.g. `hugo.toml`) were also recorded in Notion for easy reference. Markdown content in Hugo remains lightweight and clean, suited for a writing-focused blog.