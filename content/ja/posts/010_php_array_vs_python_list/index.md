+++
date = '2025-05-06T20:23:08+09:00'
draft = false
title = 'PHP array vs Python list'
+++



# 背景：

書籍では PHP の配列は演習せずに web 開発の項目に進みましたが
気になった箇所について python と比較します。

# 目的：

PHPとPythonの配列について理解する




# 1. PHPの配列は list + dict 的な性質を持つ

PHP の配列の使われ方を最初に見たとき、キーと値のペア（連想配列）的な使い方がいきなり出てきました。
リスト(list)と辞書(dict)の使い分けが特に意識されていないような印象を受けました。

一方で、Python ではリスト(list)と辞書(dict)がはっきりと区別されています。

Python に慣れていない人にとっては、こうした比較はかえって分かりづらく感じられるかもしれません。
とはいえ、ここではあくまで学習の補助や個人的な参考資料として、この比較を活用することを目的としています。




# 2. 可変性（mutability）の比較: PHP vs Python


## 2.1 Pythonにおけるmutable / immutable


| データ型     | mutable / immutable | 備考                         |
|--------------|----------------------|------------------------------|
| `int`, `str`, `tuple` | immutable             | 値の変更は不可（新規作成）     |
| `list`, `dict`, `set` | mutable               | 値を直接変更可能              |



✅ 例1: リスト（mutable）

```python
a = [1, 2, 3]
b = a  # 同じリストを参照
b.append(4)

print("a:", a)  # → a: [1, 2, 3, 4]
print("b:", b)  # → b: [1, 2, 3, 4]
```

➡️ a と b は同じリストオブジェクト。リストは mutable なので、b.append() の変更が a にも反映されます。

→ 解決方法（コピーして独立させたい場合）:


```python
a = [1, 2, 3]
b = a.copy()  # 浅いコピー（shallow copy）
b.append(4)

print("a:", a)  # → [1, 2, 3]
print("b:", b)  # → [1, 2, 3, 4]
```


✅ 例2: 数値（immutable）


```python
a = 10
b = a
b += 1

print("a:", a)  # → 10
print("b:", b)  # → 11
```


➡️ int は immutable。b += 1 は b = b + 1 の syntactic sugar であり、新しい int オブジェクトを b に再代入している。

✅ 例3: オブジェクト（mutable）


```python
class User:
    def __init__(self, name):
        self.name = name

a = User("Alice")
b = a
b.name = "Bob"

print("a.name:", a.name)  # → Bob
```


➡️ User オブジェクトは mutable。a と b は同じインスタンスを指す。


## 2.2 PHPの挙動：Copy-on-write と 参照渡し

PHP は 値渡しが基本ですが、配列やオブジェクトに関してはちょっと注意が必要です。

● 配列は「コピーされる」が、挙動が少し特殊


```php
$a = [1, 2, 3];
$b = $a;
$b[] = 4;
print_r($a);  // → [1, 2, 3]  ← a は変更されていない（コピーされている）
```

PHP の配列は一見「immutable」っぽい動きに見えます。これは**コピーオンライト（Copy-on-write）**という仕様です。
→ 同じメモリを一時的に共有していても、どちらかが書き換えると自動的にコピーが発生します。

● オブジェクトは「参照渡し」

```php
class User {
    public $name;
}
$a = new User();
$a->name = "Alice";
$b = $a;
$b->name = "Bob";
echo $a->name;  // → "Bob" ← 同じオブジェクトを指している
```


つまり、PHP 配列は「一見 immutable のような挙動」だが、オブジェクトは mutableです。


## 2.3 補足: `&`参照記号による明示的な参照


```php
$a = [1, 2];
$b =& $a;  // 参照渡し
$b[] = 3;
print_r($a);  // → [1, 2, 3]
```


# 3. 比較まとめ

✅ まとめ（比較）

| 特徴          | Python                        | PHP                                      |
|---------------|-------------------------------|-------------------------------------------|
| `list`        | mutable                       | 配列も基本は mutable（ただしコピーされる） |
| `dict`        | mutable                       | 配列で代用（連想配列）                     |
| `str`         | immutable                     | 文字列も immutable                        |
| オブジェクト   | mutable                       | 参照渡しで mutable                         |
| 特殊機構      | なし（参照は明示）            | **Copy-on-write**, `&`参照記号             |

ß
# 4. 結論

Python は明示的に「mutable/immutable」が意識される言語。

PHP は「値渡し」が基本だが、Copy-on-write や参照で裏側は mutable に近い動作をする。

アルゴリズムや設計に影響するのは、特に オブジェクトや参照を扱うときです。



