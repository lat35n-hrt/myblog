+++
date = '2025-10-25T15:34:16+09:00'
draft = false
title = 'macOS自動化処理'
categories = ["macos"]
+++



## 🪶 plist → sh → py：macOSでPythonスクリプトを自動実行する

### ✅ 目的

macOSで定期的にPythonスクリプトを実行したい場合、もっとも軽量で安定した方法が **launchd（ランチディ）** を利用する手法です。
ここでは、macOS 標準の `launchd` を使って、Pythonスクリプトを毎日自動で実行するまでの構成を説明します。

---

## 🧩 実行の流れ：plist → sh → py


```scss
launchd (.plist)
    ↓
bash script (.sh)
    ↓
Python script (.py)
```


- `.plist` … タスクスケジュールを定義する macOS 標準の設定ファイル

- `.sh` … 実行環境とログ出力を制御する中間レイヤー

- `.py` … 実際のアプリケーション処理


この3段階構成により、**安定性・再現性・保守性**が確保できます。

---

## ① plist：LaunchAgent 設定ファイル

`~/Library/LaunchAgents/com.example.daily.plist`

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
  "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>com.example.daily</string>

  <key>ProgramArguments</key>
  <array>
    <string>/Users/YourUser/scripts/daily_task.sh</string>
  </array>

  <key>StartCalendarInterval</key>
  <dict>
    <key>Hour</key><integer>20</integer>
    <key>Minute</key><integer>0</integer>
  </dict>

  <key>RunAtLoad</key><true/>
  <key>StandardOutPath</key><string>/tmp/daily_task.log</string>
  <key>StandardErrorPath</key><string>/tmp/daily_task.err</string>
</dict>
</plist>


```

### 🔍 主なポイント

|要素|説明|
|---|---|
|**Label**|macOS全体で一意となる識別名。`com.example.daily` のようにドメイン逆引き形式で管理するのが慣習。|
|**ProgramArguments**|実行するファイルパス（ここでは `.sh`）を指定。引数を配列で書くのがXML仕様。|
|**StartCalendarInterval**|実行スケジュール。ここでは「毎日20:00」。|
|**RunAtLoad**|ログイン時にも再実行可能にするオプション。|
|**StandardOutPath / StandardErrorPath**|出力とエラーをログファイルにリダイレクト。|

### 🔧 操作コマンド

```bash
launchctl load ~/Library/LaunchAgents/com.example.daily.plist
launchctl list | grep example launchctl unload ~/Library/LaunchAgents/com.example.daily.plist
```



---

## ② sh：シェルスクリプトで環境とログを整える

実際に呼び出されるのはこの `.sh` です。
ここで環境変数・PATH・仮想環境・通知などを制御します。


```bash
`#!/bin/bash export PATH=/Users/YourUser/dev/App/env/bin:/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin

cd /Users/YourUser/dev/App export

PYTHONPATH=. /Users/YourUser/dev/App/env/bin/python3

scripts/daily_summary_job.py >> /Users/YourUser/dev/App/tmp/daily.log 2>&1

osascript -e 'display notification "Job completed" with title "NewsLite"'

echo "Job completed at $(date)" >> /Users/YourUser/dev/App/tmp/daily.log echo "--"`

```

### 🧩 各ポイントの解説

|項目|理由・効果|
|---|---|
|**`which python3` で絶対パスを確認**|launchd は `$PATH` を継承しないため、Python の実行パスを明示しておくと安全。仮想環境を使う場合は `/Users/.../env/bin/python3` を直接指定。|
|**`export PATH=...`**|launchd の環境変数は最小限しか設定されないため、Homebrew・仮想環境・標準バイナリなどを明示的に指定。|
|**`cd /Users/.../App`**|カレントディレクトリを明示することで相対パスの混乱を防ぐ。|
|**`export PYTHONPATH=.`**|プロジェクト直下をモジュール検索パスに追加し、ローカルモジュールをインポート可能に。|
|**`>> ... 2>&1` によるログ統合**|標準出力と標準エラーを同一ログファイルに追記。`2>&1` は「エラー出力(2)を標準出力(1)に統合する」構文。|
|**`/tmp` ではなくプロジェクト内ログ**|`/tmp` は再起動で消えるため、`App/tmp/`に保存して履歴管理。|
|**`osascript` による通知**|GUI上で「Job completed」を表示。cronではできないlaunchd特有のUX。|
|**タイムスタンプ出力 (`date`)**|実行時刻の記録でスケジュール動作の検証が容易。|

---

### 💡 `2>&1` の補足解説

`>> logfile 2>&1` はシェル構文で、

> 「標準出力と標準エラーの両方を同じログファイルに追記する」
> という意味です。

これにより、`print()` 出力もエラーも一元的に記録され、
後からトラブル原因を確認しやすくなります。

---

## ③ py：Python スクリプト本体

この部分は任意の処理ロジックです。
例えば以下のように、日次で実行されるタスクを定義します。

```python
# scripts/daily_summary_job.py

from datetime import datetime

def main():

print(f"[{datetime.now()}] Daily job started.")    # （ここにデータ取得・整形・ファイル出力などの処理）

print("Daily job completed successfully.")

if __name__ == "__main__":
	main()

```




---

## 🧭 全体のまとめ

|レイヤー|ファイル|主な役割|
|---|---|---|
|① plist|`com.example.daily.plist`|実行スケジュールの定義（launchd）|
|② sh|`daily_task.sh`|環境・ログ・通知を制御|
|③ py|`daily_summary_job.py`|メイン処理ロジック|

このように分離しておくことで、

- システム設定（plist）

- 実行環境制御（sh）

- アプリロジック（py）
    を独立してメンテナンスでき、macOS 標準機能だけで安定した自動化が実現できます。
