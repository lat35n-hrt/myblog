+++
date = '2025-04-27T22:31:53+09:00'
draft = false
title = 'Compression Commands Cheatsheet'
+++

# File Compression & Decompression Commands Cheat Sheet

| Format | File Compression | File Decompression | Directory Compression | Directory Decompression | Key Options |
| --- | --- | --- | --- | --- | --- |
| zip | `zip archive.zip file.txt` | `unzip archive.zip` | `zip -r archive.zip dir/` | `unzip archive.zip` | `-r` (recursive for directories), `-q` (quiet mode), `-e` (set password) |
| tar (uncompressed) | `tar -cf archive.tar file.txt` | `tar -xf archive.tar` | `tar -cf archive.tar dir/` | `tar -xf archive.tar` | `-c` (create), `-x` (extract), `-f` (specify file), `-v` (verbose output) |
| tar + gzip (.tar.gz) | `tar -czf archive.tar.gz file.txt` | `tar -xzf archive.tar.gz` | `tar -czf archive.tar.gz dir/` | `tar -xzf archive.tar.gz` | `-z` (gzip compression), `--exclude=xxx` (exclude specific files) |
| tar + bzip2 (.tar.bz2) | `tar -cjf archive.tar.bz2 file.txt` | `tar -xjf archive.tar.bz2` | `tar -cjf archive.tar.bz2 dir/` | `tar -xjf archive.tar.bz2` | `-j` (bzip2 compression), `-v` (verbose output) |
| tar + xz (.tar.xz) | `tar -cJf archive.tar.xz file.txt` | `tar -xJf archive.tar.xz` | `tar -cJf archive.tar.xz dir/` | `tar -xJf archive.tar.xz` | `-J` (xz compression) |
| gzip | `gzip file.txt → file.txt.gz` | `gunzip file.txt.gz` | (Not directly available, use with tar) | (Not directly available, use with tar) | `-k` (keep original file), `-d` (decompress) |
| bzip2 | `bzip2 file.txt → file.txt.bz2` | `bunzip2 file.txt.bz2` | (Not directly available, use with tar) | (Not directly available, use with tar) | `-k` (keep original file) |
| xz | `xz file.txt → file.txt.xz` | `unxz file.txt.xz` | (Not directly available, use with tar) | (Not directly available, use with tar) | `-k` (keep original file), `-d` (decompress) |
| 7z (.7z) | `7z a archive.7z file.txt` | `7z x archive.7z` | `7z a archive.7z dir/` | `7z x archive.7z` | `-p` (set password), `-mx=9` (maximum compression) |



# Practical Use of `tar` with Compression

While `tar` can be used alone (uncompressed), in real-world practice,
it is overwhelmingly more common to use it together with a compression option.

Especially, these three formats are frequently used:

| Format | Command Example | Common Usage Reason |
| --- | --- | --- |
| .tar.gz | `tar -czf archive.tar.gz target/`<br>`tar -xzf archive.tar.gz` | The most classic choice. Fast and supported on almost any server. |
| .tar.bz2 | `tar -cjf archive.tar.bz2 target/`<br>`tar -xjf archive.tar.bz2` | Higher compression rate than `.gz`. Used when you want to minimize file size. |
| .tar.xz | `tar -cJf archive.tar.xz target/`<br>`tar -xJf archive.tar.xz` | Recently becoming more popular. Very high compression, often seen in software distribution packages. |
