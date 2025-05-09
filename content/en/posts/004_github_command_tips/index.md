+++
title = "GitHub Command Tips: Searching History, Excluding Files, Tracking Changes"
date = '2025-04-26T01:21:45+09:00'
draft = false
categories = ["GitHub"]
+++



# 💡 GitHub Command Line Tips

**Purpose:**
While working on building this blog site, I ran into a few small Git and Linux command tricks that made things more efficient — jotting them down here as a personal memo.

---

## 1️⃣ Searching Command History

It's common to revisit past commands during development. These two methods helped speed things up:

### ✅ Using `history | grep` to search

```bash
history | grep git
```


This lists all past commands containing the word git.
Great for copy-pasting and making quick edits.

###　✅ Interactive search with Ctrl + R
Pressing Ctrl + R in the terminal allows incremental reverse search.

Example: Type git, then press Ctrl + R repeatedly to cycle through previous Git-related commands.

## 2️⃣ Quickly Clear the Command Line (Ctrl + U)

Ctrl + U

This shortcut instantly clears the current command line input.


## 3️⃣ Exclude Specific Files When Using git add
If you want to run git add . but exclude specific files, you can use the :! syntax:

```bash
git add . ':!content/ja/posts/002_cloudflare_deploy_ja/index.md' ':!content/en/posts/002_cloudflare_deploy_en/index.md'
```

Use :! to explicitly exclude paths.
Because of shell behavior, it's safer to wrap paths in single quotes '.

## 4️⃣ Track File/Directory Renames with git mv
Instead of using mv, use git mv to let Git properly track renames:

```bash
git mv content/ja/posts/002_cloudflare_deploy_ja content/ja/posts/002_cloudflare_deploy
```

❗ If you use plain mv, GitHub or Cloudflare may treat the change as a "delete + add" instead of a rename.
This can break the file history and make tracking harder.

---

## ✍️ Final Thoughts
There are always new discoveries when working with Git.
Even small tips like these can significantly improve speed and reliability in your workflow.

I'll keep updating this list as I come across more useful tricks!