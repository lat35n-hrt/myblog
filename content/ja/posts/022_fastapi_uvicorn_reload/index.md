+++
date = '2025-08-19T20:49:18+09:00'
draft = false
title = 'Uvicorn コマンドの読み方を分解して理解する'
categories = ["fastapi"]
+++


FastAPI などの ASGI アプリケーションを起動するときによく使うコマンド：

```bash
uvicorn app.main:app --reload
```

一見シンプルですが、実は **2つの要素** に分けて考えると理解しやすくなります。

---

## ① `app.main` の部分

これは **Python モジュールパス** を表しています。

- `app/` ディレクトリをパッケージとして見なす

- その中の `main.py` を指定している


👉 言い換えると、**「プロジェクトルートから見た `main.py` の場所」をモジュール表記に変換したもの** です。

つまり：

```
project-root/
 └── app/
      └── main.py
```

があるときに、`app.main` と書くわけです。

---

## ② コロンの後の `app`

これは **`main.py` 内に定義された ASGI アプリケーションの変数名** を指しています。

たとえば `main.py` の中でこう書いているはずです：

```python
from fastapi import FastAPI

app = FastAPI()
```

ここで定義された **`app` という名前のオブジェクト** を、Uvicorn が ASGI アプリとしてロードします。

---

## まとめると

`uvicorn app.main:app` は、意味的にはこうなります：

- **`app.main`** = `project-root/app/main.py` モジュールを読み込む

- **`:app`** = そのモジュール内の `app` オブジェクトを使う


---

## 変数名を変えた場合

もし `main.py` に以下のように書いたとします：

```python
from fastapi import FastAPI

application = FastAPI()
```

この場合、Uvicorn を起動するコマンドはこうなります：

```bash
uvicorn app.main:application --reload
```

つまり、コロンの後ろは **ASGI アプリの変数名に合わせる必要がある** ということです。

---

## まとめ

- `app.main` = モジュールの場所 (`app/main.py`)

- `:app` = その中の ASGI アプリオブジェクトの変数名


これを理解すると、Uvicorn の起動コマンドがぐっとわかりやすくなります。