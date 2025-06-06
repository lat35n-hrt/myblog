+++
title = "GitHub操作Tips：履歴検索・除外指定・ファイル追跡など"
date = '2025-04-26T01:22:11+09:00'
draft = false
categories = ["GitHub"]
+++


# 💡 GitHub の操作 Tips

**目的：**
今回のブログサイト構築の作業中に、「これ、作業効率アップにつながるな」と感じた Git と Linux コマンドまわりのちょっとした操作Tipsをメモしておきます。

---

## 1️⃣ コマンド履歴の検索

作業中に過去のコマンドを何度も見返すことが多く、以下の方法が特に便利でした。

### ✅ `history | grep` で検索

```bash
history | grep git
```
特定のコマンドに関する履歴を一覧表示できます。コピペして再編集するのに最適。


### ✅ Ctrl + R でインタラクティブ検索
ターミナルで Ctrl + R を押すと、インクリメンタルな履歴検索が可能になります。

例：git と入力した状態で Ctrl + R を何度か押すと、過去の git コマンドが順に表示されます。

## 2️⃣ 入力中のコマンドを一気にクリア（Ctrl + U）

Ctrl + U

コマンドラインに入力中の内容を一気にクリアできます。


## 3️⃣ 一部ファイルを除外して git add
通常の git add . に特定ファイルを含めたくない場合、以下のように除外設定ができます。

```bash
git add . ':!content/ja/posts/002_cloudflare_deploy_ja/index.md' ':!content/en/posts/002_cloudflare_deploy_en/index.md'
```

:! を使って、除外したいファイル・パスを明示的に指定できます。便利ですが、シェルの構文との兼ね合いで ' で囲むのが安全です。

## 4️⃣ ファイル名・ディレクトリ変更も git mv で追跡
mv ではなく git mv を使うことで、Git に変更を正確に追跡させることができます。

```bash
git mv content/ja/posts/002_cloudflare_deploy_ja content/ja/posts/002_cloudflare_deploy
```

❗ git を使わず mv で移動すると、GitHub や Cloudflare 側では「古いファイルの削除」と「新しいファイルの追加」として扱われてしまいます。
これにより、履歴が断絶されるリスクがあります。

---

## ✍️ 最後に
Git 操作には日々発見があります。こういった小さなTipsを知っているかどうかで、
作業スピードや安全性がかなり変わってくると実感しました。

今後も便利な発見があれば追加していく予定です。