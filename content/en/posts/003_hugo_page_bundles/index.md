+++
date = '2025-04-25T09:39:13+09:00'
draft = true
title = 'Hugo Blog: How Page Bundles Work'
+++


# Understanding Page Bundles in Hugo
[Official Documentation](https://gohugo.io/content-management/page-bundles/)

---

## 📦 What Are Page Bundles?

In Hugo, you can organize a Markdown file along with its related assets (such as images or data files) in a single directory. This structure is called a **Page Bundle**.

---

## ✅ Leaf Bundle Structure (Per-Article)

Each article is managed as a separate directory, with the Markdown content written in `index.md`. Images and other assets can be placed in the same directory.

```text
content/
├── ja/
│   └── posts/
│       └── 002_cloudflare_deploy/
│           ├── index.md
│           └── image.png (optional)
├── en/
│   └── posts/
│       └── 002_cloudflare_deploy/
│           ├── index.md
│           └── image.png (optional)
```

---

🖼️ How to Insert Images
You can easily reference images located in the same directory like this:

```text
![](image.png)
```

---

🌿 When to Use Branch Bundles
By creating an _index.md file, you can customize the section (list) page for the directory it resides in.
This is useful for adding a title, description, or a table of contents to an overview or listing page.

```text
content/
└── ja/
    └── posts/
        ├── _index.md (custom section page)
        └── 001_xxx/
        └── 002_xxx/
```