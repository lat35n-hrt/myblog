+++
date = '2025-07-20T16:41:13+09:00'
draft = false
title = 'Gitトラブル対応: pushしてもGitHubに反映されない？5MB超フォントファイルを履歴ごと削除した話'
categories = ["github"]
+++


## 🧭 経緯：pushしたのに反映されない？

ある日、普段通りに `git add` → `commit` → `push` を6回ほど繰り返した後、GitHubを確認してみると、更新が**反映されていない**ことに気づきました。

最初は単なるタイムラグかと思いましたが、何度試してもダメ。`push` にも特にエラーは出ておらず、**ローカルでは最新の履歴になっている**のに、GitHub側では一向に更新されません。

原因を探るうちに、あるファイルが浮上しました。

---

## 🧱 問題の原因：フォントファイル（5.5MB）

履歴に含まれていた `fonts/NotoSansJP-Regular.ttf`（約5.5MB）が原因で、GitHubへの push が正常に行われていなかった可能性が高いと判断。

Gitの挙動やエラーが明確でない場合、このような**バイナリ大容量ファイルが履歴に含まれていること**が障害になることがあります。

---

## 🔍 各コマンドの目的と効果の分析

### ✅ 1. `brew install git-filter-repo`

- **目的**：Git履歴の操作ツール `git-filter-repo` を導入

- **効果**：旧式の `git filter-branch` よりも高速・安全にファイル削除が可能


> ※macOS環境では `brew` が便利です。

---

### ✅ 2. `git filter-repo --path fonts/NotoSansJP-Regular.ttf --invert-paths`

- **目的**：フォントファイルを**履歴ごと完全に削除**

- **効果**：

    - 現在のコミットだけでなく、**過去の全履歴**からファイルを抹消

    - `.git/objects` 内からも削除対象として再構築される


---

### ✅ 3. `git gc --prune=now --aggressive`

- **目的**：削除されたオブジェクトを**完全に消去**

- **効果**：

    - `.git/objects` のクリーンアップ

    - リポジトリの容量削減、ゴミの除去


---

### ✅ 4. `git config --global http.postBuffer 524288000`

- **目的**：push時の**バッファ制限を500MBに拡張**

- **効果**：

    - 「RPC failed」や「error: 400」系の push エラーを防止

    - 通信品質が不安定な環境や大容量 push にも耐性向上


---

### ✅ 5. `git push --force origin main`

- **目的**：ローカルの書き換え済履歴をリモートに**強制反映**

- **効果**：

    - GitHub上の履歴も filter 後の状態に置き換えられる

    - ※チーム開発中は慎重に（今回は自分だけのスタック状態だったため問題なし）


---

## ✅ 結果：どうなったか

|項目|状況|
|---|---|
|フォントファイル（5.5MB）|**履歴ごと削除完了**（filter-repoで完全除去）|
|`.gitignore` による予防|`fonts/*.ttf` を追加し、**今後は含まれないよう設定**|
|GitHub 上の履歴|**クリーンな状態に更新**され、無事 push 成功|
|push 時のエラー|`http.postBuffer` 拡張により**回避済み**|
|今後の方針|READMEなどで**外部からフォントをダウンロード案内**に切り替える予定|

---

## 📝 教訓：Git履歴にバイナリは入れない方がいい

今回のようなトラブルを防ぐためにも、以下を意識したいところです。

- `.gitignore` を活用し、最初からバイナリは除外する

- フォントや画像などの大容量ファイルは CDN や外部リンクに切り出す

- 誤って追加した場合は `git-filter-repo` で履歴ごと削除する手段を覚えておく


---

## 💬 最後に

「push しても GitHub に反映されない」という事象は、Git を使っていて一度は直面するかもしれません。今回のように大容量ファイルの混入が原因の場合、エラーが明確に出ない分、気付きづらいものです。

困ったら、冷静に `git log` や `.gitignore` を見直し、**履歴ごと削除して仕切り直す**ことで、意外とスッキリ解決することもあります。