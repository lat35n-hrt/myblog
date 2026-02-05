+++
date = '2026-02-05T15:33:13+09:00'
draft = false
title = 'Pythonの正規表現で文字列クリーニング'
categories = ["python"]
+++

テキストデータの前処理では、不要な文字の除去や
パターンマッチングが頻繁に発生します。
Pythonの`re`モジュールを使った実践的なパターンを紹介します。

## 1. re.sub() - 文字列の置換・クリーニング

### 基本構文
```python
import re
re.sub(pattern, replacement, string)
```

- **pattern**: 検索する正規表現パターン
- **replacement**: 置換後の文字列
- **string**: 対象の文字列

### 実践例：特定文字以外を削除
```python
text = "hello, world! don't stop #learning"
cleaned = re.sub(r"[^a-zA-Z0-9'\s]", ' ', text)
print(cleaned)
# "hello  world  don't stop  learning"
```

### パターンの解説
```
r"[^a-zA-Z0-9'\s]"

r""         : raw文字列（\を特殊扱いしない）
[]          : 文字クラス（文字の集合を定義）
^           : 否定（[]内の先頭で使用時）
a-zA-Z      : すべての英字
0-9         : すべての数字
'           : アポストロフィ（そのまま）
\s          : 空白文字（スペース、タブ、改行など）

→ 「英数字、アポストロフィ、空白以外」をスペースに置換
```

### ユースケース
- ユーザー入力のサニタイズ
- ファイル名の正規化
- テキストマイニングの前処理
- ログデータのクリーニング

---

## 2. re.search() - パターンの検索

### 基本構文
```python
import re
re.search(pattern, string)
```

マッチする部分が見つかれば`Match`オブジェクト、
見つからなければ`None`を返す。

### 実践例：英数字の存在チェック
```python
# 単語に英数字が含まれているか確認
word = "don't"
if re.search(r'[a-zA-Z0-9]', word):
    print("有効な単語")
else:
    print("無効な単語（記号のみ）")
```

### 応用：アポストロフィのみの単語を除外
```python
words = ["hello", "world", "'", "don't", "''"]
valid_words = [w for w in words if re.search(r'[a-zA-Z0-9]', w)]
print(valid_words)
# ['hello', 'world', "don't"]
```

### ユースケース
- 入力バリデーション
- フィルタリング処理
- 条件分岐の判定
- データクレンジング

---

## 3. 実務での組み合わせ例

### テキスト分析パイプライン
```python
import re

def clean_and_filter(text):
    """テキストをクリーニングし、有効な単語のみ抽出"""
    # 1. 特殊文字を削除
    cleaned = re.sub(r"[^a-zA-Z0-9'\s]", ' ', text)

    # 2. 単語に分割
    words = cleaned.split()

    # 3. 英数字を含む単語のみフィルタリング
    valid_words = [w for w in words if re.search(r'[a-zA-Z0-9]', w)]

    return valid_words

# 使用例
text = "hello, world! don't stop. ''' #hashtag"
result = clean_and_filter(text)
print(result)
# ['hello', 'world', "don't", 'stop', 'hashtag']
```

### ログファイルの解析
```python
def extract_error_lines(log_text):
    """エラー行のみ抽出"""
    lines = log_text.split('\n')
    error_lines = [line for line in lines
                   if re.search(r'ERROR|CRITICAL', line)]
    return error_lines
```

---

## 4. よく使う正規表現パターン集

| パターン | 意味 | 用途 |
|---------|------|------|
| `[^...]` | 否定（...以外） | 不要文字の削除 |
| `\s` | 空白文字 | スペース・タブ・改行 |
| `\d` | 数字（0-9） | 数値の検出 |
| `\w` | 単語文字（a-zA-Z0-9_） | 識別子の検証 |
| `+` | 1回以上の繰り返し | 連続パターン |
| `*` | 0回以上の繰り返し | 省略可能パターン |


---

## まとめ

- **re.sub()**: 文字列の置換・クリーニングに使用
- **re.search()**: パターンの存在確認に使用
- **組み合わせ**: 実務的なテキスト処理パイプラインを構築可能


つい文字列操作で個別対応してしまいそうになりますが、
正規表現を利用するとシンプルに対応できることが多いですね。


---

## 参考リンク
- [Python re モジュール公式ドキュメント](https://docs.python.org/ja/3/library/re.html)
- [正規表現のテストツール regex101](https://regex101.com/)
```

---


