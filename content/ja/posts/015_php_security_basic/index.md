+++
date = '2025-05-13T09:46:10+09:00'
draft = false
title = 'PHP セキュリティ基礎'
+++




PHPとセキュリティ：最低限押さえておきたい3つの対策
WebアプリケーションをPHPで開発する際、セキュリティ対策は避けて通れません。セキュリティの穴を放置すると、個人情報の漏洩やサイトの改ざんといった深刻な被害に繋がる可能性があります。

本記事では、PHPで必ず押さえておきたい代表的なセキュリティ対策として、以下の4つを紹介します。

- XSS（クロスサイトスクリプティング）対策

- SQLインジェクション対策

- パスワードの安全な保存

- CSRF対策

<br>

## 1. XSS対策：htmlspecialchars()
◆ XSSとは？
XSS（クロスサイトスクリプティング）は、悪意のあるスクリプトをWebページ上に埋め込み、ユーザーのブラウザで実行させる攻撃です。掲示板やコメント欄など、ユーザーの入力をそのままHTMLに出力する場合に危険です。

◆ 対策方法
PHPでは htmlspecialchars() を使って、HTMLタグなどの特殊文字をエスケープすることで、XSSを防ぐことができます。

```php
$user_input = $_GET['comment'];
$safe_output = htmlspecialchars($user_input, ENT_QUOTES, 'UTF-8');
echo $safe_output;
```

◆ ポイント
第2引数に ENT_QUOTES を指定すると、シングルクオートも変換対象になります。

第3引数で文字コードを明示的に指定しましょう（例：UTF-8）。



<br>

## 2. SQLインジェクション対策：プリペアドステートメント（PDO）
◆ SQLインジェクションとは？
SQLインジェクションは、ユーザーの入力内容にSQL構文を混ぜることで、意図しないクエリを実行させる攻撃です。ログイン処理など、データベースに対してクエリを投げる処理で特に注意が必要です。

◆ 対策方法
PDO（PHP Data Objects）のプリペアドステートメントを使えば、プレースホルダで値を安全に埋め込むことができ、SQLインジェクションを防げます。


```php
$pdo = new PDO('mysql:host=localhost;dbname=testdb;charset=utf8', 'user', 'password');
$stmt = $pdo->prepare('SELECT * FROM users WHERE email = :email');
$stmt->bindValue(':email', $email, PDO::PARAM_STR); // その場で値をバインド
$stmt->execute();

```

◆ ポイント
- 値をSQL文に直接連結しないこと。

- プレースホルダ（? や :name）を使い、基本は bindValue() を推奨。

- bindParam() は変数への参照を渡すため、ループなどで扱いに注意が必要。



<br>

## 3. パスワードの安全な保存：password_hash() / password_verify()
◆ なぜそのまま保存してはいけないのか？
パスワードを平文（そのままの文字列）でデータベースに保存すると、情報漏洩時に大きな被害を受けます。ハッシュ化して保存することで、漏洩時のリスクを軽減できます。

◆ 対策方法
PHPには安全なパスワードハッシュ化のための関数が標準で用意されています。

登録時（ハッシュ化）

```php
$hash = password_hash($password, PASSWORD_DEFAULT);

```

認証時（照合）

```php
if (password_verify($input_password, $hash_from_db)) {
    echo "ログイン成功";
} else {
    echo "ログイン失敗";
}

```


◆ ポイント
PASSWORD_DEFAULT を使うことで、PHPが推奨する安全なアルゴリズム（現在はbcrypt）が使われます。

password_hash() はソルトも自動で付与してくれるため、個別に生成する必要はありません。



<br>

## 4. CSRF対策：トークンによるリクエスト検証
◆ CSRFとは？
CSRF（クロスサイトリクエストフォージェリ）は、ログイン中のユーザーの権限を悪用して、意図しない操作を実行させる攻撃です。たとえば、本人が意図していないのに「退会処理」や「送金」ボタンが押されてしまうといったケースが該当します。

◆ 対策方法
CSRFトークン（ワンタイムトークン） を発行してフォームに埋め込み、サーバー側で一致を確認することで、第三者の不正リクエストを防ぎます。

① フォーム生成時にトークンを埋め込む

```php
session_start();
if (empty($_SESSION['token'])) {
    $_SESSION['token'] = bin2hex(random_bytes(32));
}
?>

<form method="post" action="submit.php">
  <input type="hidden" name="csrf_token" value="<?php echo htmlspecialchars($_SESSION['token'], ENT_QUOTES, 'UTF-8'); ?>">
  <!-- 他の入力フィールド -->
  <input type="submit" value="送信">
</form>

```

② サーバー側でトークンを検証

```php
session_start();

if (!isset($_POST['csrf_token']) || $_POST['csrf_token'] !== $_SESSION['token']) {
    die('不正なリクエストです');
}

// 正常な処理を続行

```


<br>

◆ ポイント<br>
トークンはセッションごと、またはフォームごとに生成するのが理想です。

JavaScriptで生成・送信されるフォームにもCSRF対策が必要です（例：Ajax + Cookieベース認証の場合など）。


✅ 補足：CSRFトークンは使い回しせず、処理後に unset($_SESSION['token']) などで破棄しましょう。また、セッション固定攻撃の対策としてログイン直後に session_regenerate_id(true) を実行することも推奨されます。


<br>

## まとめ

| 脅威          | 対策                                      |
| ----------- | --------------------------------------- |
| XSS         | `htmlspecialchars()` による出力時エスケープ        |
| SQLインジェクション | `PDO` + プリペアドステートメント                    |
| パスワード漏洩     | `password_hash()` / `password_verify()` |
| CSRF        | トークンの埋め込みと検証                            |
