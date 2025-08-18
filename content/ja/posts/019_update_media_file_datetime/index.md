+++
date = '2025-07-08T19:41:32+09:00'
draft = false
title = 'MP4の撮影日時をあとから一括修正した話【ExifとFinderの作成日を揃える】'
categories = ["macos"]
+++



## 環境
- Mac OS: Venture 13.7.2

- カメラ：DJI Pocket 2

- SD カード： SanDisk Extreme 256GB microSD SDXC UHS-1 U3 V30


## 🎯 なぜやるのか？

大阪万博で撮影した動画の**日時設定が初期状態（デフォルト）**のままでした。
これでは整理や検索ができずに困るため、**ExifヘッダーとFinder上の「作成日」「変更日」**を修正することにしました。

> ⚠️ 本作業は**個人利用**に限定しています。また、すべての日時情報を変更できたわけではないことも、あわせて共有しておきます。

---

## 1. Exifヘッダーの日時を一括修正する

ExifToolを使えば、**`DateTimeOriginal`（撮影日時）/ `CreateDate` / `ModifyDate`**をまとめて一括修正できます。

### 📌 コマンド例：

````bash
exiftool "-AllDates+=9:5:1 13:17:27" -overwrite_original /Volumes/DCIM/100MEDIA/*.MP4

````


- `-AllDates+=9:5:1 13:17:27`
    → **9年5ヶ月1日と13時間17分27秒**を加算してシフト

- `-overwrite_original`
    → 元ファイルを上書き保存


### ✅ この操作で変わるExifフィールド：

- `DateTimeOriginal`

- `CreateDate`

- `ModifyDate`


### ⚠️ 注意：

Finder上の**「作成日」「変更日」には反映されません**。

---

## 2. Finderで見える「作成日」「変更日」を変更する

ExifToolを使って、ファイルシステム上の日時も修正可能です。

### 📌 コマンド例：

````bash
exiftool -P -overwrite_original \
   "-FileModifyDate<CreateDate" \
   "-FileCreateDate<CreateDate" \
   /Volumes/Untitled/DCIM/100MEDIA/*.MP4
````

- `-P`：オリジナルのファイルアクセス権を維持

- `<CreateDate`：Exif内の `CreateDate` を基準にコピー


### ⚠️ `setfile` エラー対策

この操作には macOS の `setfile` コマンドが必要です。
**Xcode Command Line Tools** が未インストールの場合、以下で導入：

````bash
xcode-select --install
````


---

## 3. メタデータを確認するには？

対象ファイルの**すべての時間情報**を確認するには、以下のコマンドが便利です。

````bash
exiftool -time:all -a -G1 -s /Volumes/Untitled/DCIM/100MEDIA/test/DJI_0215.MP4

````


### 🧾 出力例（抜粋）：

````yaml

[System] FileModifyDate     : 2025:06:09 08:01:14+09:00
[QuickTime] CreateDate      : 2025:06:09 08:01:14
[Track1] MediaCreateDate    : 2016:01:07 18:43:47
[Track2] MediaCreateDate    : 2016:01:07 18:43:47  ...

````



---

## 🚫 なぜ `TrackCreateDate` や `MediaCreateDate` は変更できないのか？

以下のフィールドは変更が困難、あるいは非対応です：

- `[Track1]〜[Track4]` の `TrackCreateDate`, `MediaCreateDate` など


### ❓理由：

- これらはMP4ファイル内部の**バイナリ構造（moov atom）**に埋め込まれており、**ExifToolでは書き換え非対応**です。

- **複数トラック構成**の動画では、正確な再構築には**専門的なビデオ編集ツール**が必要です。


---

## ✍️ まとめ

|項目|状態|
|---|---|
|Exifの撮影日時|✅ 修正可能（`AllDates`）|
|Finderの作成日/変更日|✅ 修正可能（Xcodeツール必要）|
|Track系の内部日時|❌ 書き換え不可（バイナリ構造）|

ExifToolは非常に強力ですが、**MP4のトラックレベルの日時編集**までは対応していません。とはいえ、**実用上はExifとファイル日時の統一で十分**なケースも多いため、まずはこの手順で整理してみてはいかがでしょうか。