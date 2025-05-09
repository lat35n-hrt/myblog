+++
date = '2025-04-27T22:31:40+09:00'
draft = false
title = '圧縮コマンド チートシート'
categories = ["Linux"]
+++

# ファイル圧縮・解凍コマンド チートシート

| 圧縮形式 | ファイル圧縮 | ファイル解凍 | ディレクトリ圧縮 | ディレクトリ解凍 | 主なオプション |
| --- | --- | --- | --- | --- | --- |
| zip | `zip archive.zip file.txt` | `unzip archive.zip` | `zip -r archive.zip dir/` | `unzip archive.zip` | `-r` (ディレクトリ圧縮), `-q` (静かに実行), `-e` (パスワード設定) |
| tar (非圧縮) | `tar -cf archive.tar file.txt` | `tar -xf archive.tar` | `tar -cf archive.tar dir/` | `tar -xf archive.tar` | `-c` (作成), `-x` (展開), `-f` (ファイル指定), `-v` (詳細表示) |
| tar + gzip (.tar.gz) | `tar -czf archive.tar.gz file.txt` | `tar -xzf archive.tar.gz` | `tar -czf archive.tar.gz dir/` | `tar -xzf archive.tar.gz` | `-z` (gzip圧縮), `--exclude=xxx` (除外) |
| tar + bzip2 (.tar.bz2) | `tar -cjf archive.tar.bz2 file.txt` | `tar -xjf archive.tar.bz2` | `tar -cjf archive.tar.bz2 dir/` | `tar -xjf archive.tar.bz2` | `-j` (bzip2圧縮), `-v` (進捗表示) |
| tar + xz (.tar.xz) | `tar -cJf archive.tar.xz file.txt` | `tar -xJf archive.tar.xz` | `tar -cJf archive.tar.xz dir/` | `tar -xJf archive.tar.xz` | `-J` (xz圧縮) |
| gzip | `gzip file.txt → file.txt.gz` | `gunzip file.txt.gz` | (直接不可 / tarと併用) | (直接不可 / tarと併用) | `-k` (元ファイルを保持), `-d` (解凍) |
| bzip2 | `bzip2 file.txt → file.txt.bz2` | `bunzip2 file.txt.bz2` | (直接不可 / tarと併用) | (直接不可 / tarと併用) | `-k` (元ファイルを保持) |
| xz | `xz file.txt → file.txt.xz` | `unxz file.txt.xz` | (直接不可 / tarと併用) | (直接不可 / tarと併用) | `-k` (元ファイルを保持), `-d` (解凍) |
| 7z (.7z) | `7z a archive.7z file.txt` | `7z x archive.7z` | `7z a archive.7z dir/` | `7z x archive.7z` | `-p` (パスワード設定), `-mx=9` (最大圧縮) |



# 圧縮オプション付き `tar` の実用例

`tar` は単体（非圧縮）でも使えますが、実務では
圧縮オプションと組み合わせて使うことが圧倒的に多いです。

特に頻繁に使われるのはこの3つです：

| 形式 | コマンド例 | よく使われる理由 |
| --- | --- | --- |
| .tar.gz | `tar -czf archive.tar.gz target/`<br>`tar -xzf archive.tar.gz` | 超定番。高速で、ほぼすべてのサーバでサポートされている。 |
| .tar.bz2 | `tar -cjf archive.tar.bz2 target/`<br>`tar -xjf archive.tar.bz2` | `.gz` よりも圧縮率が高い。ファイルサイズをできるだけ小さくしたいときに使用。 |
| .tar.xz | `tar -cJf archive.tar.xz target/`<br>`tar -xJf archive.tar.xz` | 最近増えている形式。非常に高圧縮で、ソフトウェア配布パッケージによく使われる。 |
