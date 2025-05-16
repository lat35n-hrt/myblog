+++
date = '2025-05-16T16:40:45+09:00'
draft = false
title = 'PHP x SQL 動的クエリ'
categories = ["sql"]
+++


# 【PHP×SQL】動的にWHERE句を追加する定番テクニック「1=1」の使い方とは？

Webアプリケーションを作っていると、「ユーザーが指定した条件に応じてSQLを柔軟に組み立てたい」という場面によく出会います。特に検索画面やフィルター機能では、条件が複数あり、かつすべてが任意入力であるケースが多いですよね。

そんなときによく使われるテクニックが、**`WHERE 1=1`** というおまじないのような書き方です。この記事では、その理由と使い方を、実例を交えて解説します。


## ■ 1=1 を使う理由とは？

SQLでは、`WHERE`句に条件を追加していく際に、文字列を連結して動的に組み立てることがあります。

しかし、条件を複数追加していくと、以下のように先頭に`AND`を付けるかどうかを都度考慮する必要が出てきます。

```php
$sql = 'SELECT * FROM books WHERE';
if ($minPrice !== null) {
    $sql .= ' price >= :min_price';
}
if ($maxPrice !== null) {
    $sql .= ' AND price <= :max_price';
}


```

このように記述すると、最初の条件に`AND`をつけるかどうか、条件がいくつあるかを意識する必要があり、非常に煩雑です。

そこで登場するのが `1=1` のテクニックです。

```sql
WHERE 1=1

```

は常に真（TRUE）になる式で、**「ダミーの最初の条件」として使われる**ことで、以降の条件に無条件で `AND` を付けられるようになります。


## ■ 実例：価格帯と出版年で検索する本の一覧

以下のPHPコードでは、`$minPrice`、`$maxPrice`、`$year` という3つのオプション条件を使って、本の一覧を取得するSQLを組み立てています。

```php
// SQLを動的に構築
$sql = 'SELECT * , is_borrowed FROM books WHERE 1=1';

$params = [];

if ($minPrice !== null) {
    $sql .= ' AND price >= :min_price';
    $params[':min_price'] = $minPrice;
}

if ($maxPrice !== null) {
    $sql .= ' AND price <= :max_price';
    $params[':max_price'] = $maxPrice;
}

if ($year !== null) {
    $sql .= ' AND YEAR(publish) = :year'; // MySQLのYEAR()関数
    $params[':year'] = $year;
}

```


これにより、例えば `$minPrice` と `$year` だけが指定されていた場合、SQLは以下のようになります：

```sql
SELECT * , is_borrowed FROM books
WHERE 1=1
  AND price >= :min_price
  AND YEAR(publish) = :year
```


`1=1` を使うことで、**`AND` をすべての条件につけて問題なし**という状態が作れるため、コードがとてもシンプルになります。

---

## ■ まとめ：1=1 は“読みやすさ”と“柔軟性”を両立する

`WHERE 1=1` は、あくまで動的SQLを安全・簡潔に組み立てるためのテクニックであり、本番コードでも多く使われています。

この書き方には以下のようなメリットがあります：

- 条件の有無を気にせず `AND` を使える

- 条件の順番や数が変わってもロジックが壊れにくい

- SQLの組み立てが読みやすく、保守性が高い


一見「意味のない条件」に見えるかもしれませんが、**動的SQLを扱う開発者にとっては“便利なダミー”**として役立ちます。