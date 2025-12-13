+++
date = '2025-12-13T15:26:12+09:00'
draft = false
title = 'GitHub履歴からwrangler.tomlを削除'
categories = ["github"]
+++


## この記事を書く理由

作業中にGPTから「Cloudflare Workers の KV ID を GitHub に漏洩するとアカウント停止ペナルティを受ける」という警告を受け、慌てて対応を実施しました。

後日、複数の情報源で確認したところ、この警告は **ハルシネーション（誤情報）** であることが判明しました。KV namespace ID は識別子であり、API Key のような機密情報ではありません。

しかし、以下の理由から作業は無駄ではなかったと考えています：

1. **ベストプラクティスの実践**: 設定情報をハードコーディングせず、サンプルファイルとして管理する運用に移行できた
2. **git-filter-repo の学習**: 履歴書き換えツールの使い方と注意点を実践的に学べた
3. **エラー対応の記録**: `fresh clone` 必須というエラーに遭遇し、その解決方法を記録として残せる

この記事では、焦りから始まった作業の経緯と、正しい技術的手順を記録します。


## 背景

Cloudflare Workers の `wrangler.toml` を GitHub に commit してしまっていたため、履歴から完全に削除する作業を実施しました。


## 環境
- **Git**: 2.39.2
- **git-filter-repo**: 最新版（2025年12月時点）
  - GitHubリポジトリから直接インストール


### 削除対象の情報

toml

```toml
[[kv_namespaces]]
binding = "MY_KV"
id = "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
preview_id = "yyyyyyyyyyyyyyyyyyyyyyyyyyyy"
```

**重要な補足**：これらのKV namespace IDは**識別子であり、API Keyではありません**。Cloudflare公式ドキュメントでも「public」として扱われています。しかし、設定情報をハードコーディングしないというベストプラクティスの観点から削除を決定しました。

---

## git-filter-repo を使った履歴削除手順

### 前提条件

bash

```bash
# git-filter-repo のインストール確認
pip install git-filter-repo
# または
brew install git-filter-repo
```

### STEP 1: 作業ディレクトリのバックアップ

bash

```bash
mv newslite-ui newslite-ui-backup
```

**理由**：git-filter-repo は破壊的な操作を行うため、万が一の復元に備える

---

### STEP 2: 新規クローンの作成

bash

````bash
git clone https://github.com/myaccount/newslite-ui.git newslite-ui-clean
cd newslite-ui-clean
```

**重要**：git-filter-repo は「fresh clone」でのみ実行可能です。既存の作業ディレクトリでは以下のエラーが発生します：
```
Aborting: Refusing to destructively overwrite repo history since
this does not look like a fresh clone.
````

これは安全機構であり、手元の `.git/refs` や作業状態が破壊されるのを防ぐための仕様です。

---

### STEP 3: 履歴からファイルを削除

bash

```bash
git filter-repo --path wrangler.toml --invert-paths
```

**オプション説明**：

- `--path wrangler.toml`：対象ファイルを指定
- `--invert-paths`：指定したファイルを削除し、それ以外を残す

この操作により、全コミット履歴から `wrangler.toml` が削除されます。

---

### STEP 4: GitHubへ強制プッシュ

bash

```bash
git push origin main --force
```

⚠️ **注意**：`--force` は既存のリモート履歴を上書きする破壊的操作です。チーム開発の場合は事前に周知が必要です。

---

### STEP 5: 設定ファイルの再配置

bash

```bash
# サンプルファイルを作成
touch wrangler.sample.toml

# .gitignore に "wrangler.toml" を追加
echo "wrangler.toml" >> .gitignore
git add .gitignore wrangler.sample.toml
git commit -m "Add wrangler.sample.toml and ignore wrangler.toml"
git push origin main
```

今後は `wrangler.sample.toml` をテンプレートとしてGitHub共有し、実際の `wrangler.toml` は引き続き使用する運用にしました。