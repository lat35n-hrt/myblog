+++
date = '2025-10-17T21:16:57+09:00'
draft = false
title = "if __name__ == '__main__': の基本"
categories = ["python"]
+++


Python ではファイルを直接実行したときだけ動かしたいコードを

`if __name__ == "__main__":`

ブロックに書くのが一般的です。

```python
def fetch_articles(debug=False):
    if debug:
        print("Debug mode: fetching mock articles...")
        return [{"title": "Test Article"}]
    # 実際の処理 ...
    pass


if __name__ == "__main__":
    articles = fetch_articles(debug=True)
    print(articles)
```


## 💡 どういうときに使う？
- スクリプトを直接実行してデバッグしたいとき

- 他のモジュールから import されたときに余計な処理を走らせたくないとき

- 単体実行（簡易テスト）と再利用の両立をしたいとき

## 🧭 挙動のまとめ

|実行方法|`__name__` の値|ブロックは実行される？|
|---|---|---|
|`python script.py`|`"__main__"`|✅ 実行される|
|`import script`|`"script"`|❌ 実行されない|


## 🧠 使いどころ
開発中は下記のようにしておくと便利です。

```python
if __name__ == "__main__":
    # 単体テストや動作確認
    articles = fetcharticles(debug=True)
```

- デバッグ時に debug=True で動作を確認できる

- pytest や import 時にはこの部分が実行されないため安全

## ✏️ まとめ
if __name__ == "__main__": は、
「このファイルを直接実行したときだけ動かしたい処理」を切り分けるための定番構文です。
デバッグや単体利用の入り口として非常に便利です。