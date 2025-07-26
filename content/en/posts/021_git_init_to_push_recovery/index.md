+++
date = '2025-07-26T23:28:03+09:00'
draft = false
title = 'A Complete Guide to Undoing Git Mistakes: init → add → commit → push'
categories = ["github"]
+++



Git is a powerful tool, but it's easy to make mistakes—like running a command too early or in the wrong place.
This guide walks through how to recover from common missteps during the basic Git workflow: `init → add → commit → push`.

---

## 1. Mistaken `git init`: Initialized in the Wrong Directory

### ❌ The Problem
You ran `git init` in a directory that shouldn't be a Git repository (e.g., a parent folder by mistake).

### ✅ The Fix
You can remove Git tracking with the following command:

```bash
rm -rf .git
```


This removes the .git directory, and the folder is no longer tracked by Git.
Navigate to the correct project directory and run git init again there.

## 2. Premature git add: Still Editing or Forgot to Save
### ❌ The Problem
You staged files with git add, but you're not done editing—or you forgot to save changes first.

### ✅ The Fix
To unstage a specific file:

```bash
git reset <filename>
```

To unstage everything:

```bash
git reset
```

The file contents are not affected. You can continue editing and then stage them again when ready.

## 3. Already Committed? Fixing git commit Mistakes
### ❌ The Problem
You made a commit too soon, or the commit message contains a typo. Maybe you included something you didn’t mean to.

### ✅ Fixes by Situation

| Goal                          | Command                   | Keeps Changes | Staged? |
| ----------------------------- | ------------------------- | ------------- | ------- |
| Completely discard the commit | `git reset --hard HEAD~1` | ❌ No          | ❌ No    |
| Keep changes and stay staged  | `git reset --soft HEAD~1` | ✅ Yes         | ✅ Yes   |
| Keep changes and unstage them | `git reset HEAD~1`        | ✅ Yes         | ❌ No    |



### ✅ Just Fix the Commit Message

```bash
git commit --amend
```

This opens your default editor (like Vim) to edit the message.
The commit content stays the same unless you modify it during the amend.

## 4. Pushed Something Sensitive? Cleaning Up After git push
### ❌ The Problem
You forgot to include a file in .gitignore, staged it with git add ., and then pushed it.
It turns out that file contains API keys, credentials, or other sensitive data.

### ✅ The Fix: Remove from Entire History
Use git-filter-repo to delete the file from all Git history:

``` bash
git filter-repo --path secrets.env --invert-paths
```

Then force-push to update the remote repository:

```bash
git push origin main --force
```

### ⚠️ Be very careful with --force if you're working on a team. Communicate and back up before proceeding.

## 📝 Summary
In the Git workflow init → add → commit → push, mistakes are common—but they’re usually recoverable.

Thankfully, Git offers robust tools to undo or fix mistakes at every step.

Knowing how to recover confidently means you can work without fear—even if something goes wrong.