+++
date = '2025-07-20T16:41:29+09:00'
draft = false
title = 'Git Troubleshooting: Push Didn’t Show Up on GitHub? How I Removed a 5MB Font File from Git'
categories = ["github"]
+++



## 🧭 Background: Pushed, but Nothing Appeared on GitHub?

After performing `git add` → `commit` → `push` about six times as usual, I noticed something odd: **my updates weren’t showing up on GitHub**.

At first, I suspected a delay or caching issue—but no matter how many times I pushed, GitHub still didn’t reflect the changes.

No errors occurred during the push, and everything looked up-to-date on my local branch. After some digging and asking ChatGPT for help, I found the likely culprit.

---

## 🧱 The Cause: A 5.5MB Font File

A binary file named `fonts/NotoSansJP-Regular.ttf` (about 5.5MB) had made its way into the commit history. While Git can technically handle large files, they often **cause silent push failures** or hang-ups—especially when combined with shallow clones or GitHub’s backend limitations.

This file had to go—not just from the current commit, but from the entire history.

---

## 🔍 Breakdown of Each Command and Its Purpose

### ✅ `brew install git-filter-repo`
- **Purpose**: Install the `git-filter-repo` tool.
- **Effect**: Replaces the older `git filter-branch` tool. Much faster and more reliable for rewriting Git history.
- **Why**: The official recommended method for removing files from Git history.

---

### ✅ `git filter-repo --path fonts/NotoSansJP-Regular.ttf --invert-paths`
- **Purpose**: Remove the font file from **the entire Git history**.
- **Effect**:
  - Rewrites the history as if the file never existed.
  - Effectively removes the file from `.git/objects`.

---

### ✅ `git gc --prune=now --aggressive`
- **Purpose**: Clean up unused Git objects, especially those removed by `filter-repo`.
- **Effect**:
  - Deletes orphaned or dangling objects.
  - Shrinks repo size and removes lingering traces of large files.

---

### ✅ `git config --global http.postBuffer 524288000`
- **Purpose**: Increase Git’s HTTP post buffer to 500MB.
- **Effect**:
  - Prevents push failures like `RPC failed` or `curl 22 The requested URL returned error: 400`.
  - Useful for large pushes or unreliable network conditions.

---

### ✅ `git push --force origin main`
- **Purpose**: Force-push the rewritten history to GitHub.
- **Effect**:
  - Overwrites the remote history with your cleaned local version.
  - Use caution in team settings (in this case, safe because it was a solo project).

---

## ✅ Outcome Summary

| Item                             | Status                                                        |
|----------------------------------|---------------------------------------------------------------|
| Font file (5.5MB)                | **Removed completely** from history using `filter-repo`       |
| Preventing future issues         | Added `fonts/*.ttf` to `.gitignore` to avoid re-adding        |
| GitHub repository history        | **Successfully rewritten** with a clean state                 |
| Push errors                      | **Resolved** by increasing `http.postBuffer` limit            |
| Recommended future practice      | Use README to instruct users to **download fonts externally** |

---

## 📝 Key Takeaways

To avoid similar issues in the future:

- Use `.gitignore` early to keep binaries out of your repo.
- Host large files externally (e.g., fonts, media, assets).
- If large files do get committed, learn how to remove them with `git-filter-repo`.

---

## 💬 Final Thoughts

When your `push` doesn’t reflect on GitHub with no errors, large binary files in your history might be to blame. Since Git doesn’t always warn you clearly, these issues can be subtle and frustrating.

Fortunately, tools like `git-filter-repo` make it easy to clean up your repo. Once cleaned, you’ll likely notice smaller repo sizes, faster clones, and fewer push issues.

If you're facing a similar situation—don't panic. Clean the history, add some preventive `.gitignore` rules, and keep moving forward.
