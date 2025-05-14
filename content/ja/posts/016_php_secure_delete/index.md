+++
date = '2025-05-14T19:31:23+09:00'
draft = false
title = 'PHP 削除機能実装(セキュリティ対策)'
categories = ["PHP"]
+++



### ✅ 削除機能の実装ポイント（セキュリティ対策含む）

前回の記事では「CSRFトークン」や「認証チェック」の実装について紹介しました。今回はその応用として、**削除機能を自前で実装**した際に注意したセキュリティ上のポイントをまとめます。

---

<br>

### 1. 🔐 **CSRFトークンによるリクエスト検証**

```php

if (empty($_POST['token']) || !hash_equals($_SESSION['token'], $_POST['token'])) {
    echo "Invalid CSRF token.";
    exit;
}

```


外部サイトからユーザーになりすまして送られる悪意あるリクエストをブロックするために、**セッションとPOSTデータのトークンが一致するかを検証**しています。これは前回紹介したCSRF対策の実践例です。

<br>

### 2. 🚫 **GETではなくPOSTメソッドを使用**


```php

if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    echo "Invalid request method.";
    exit;
}

```

<br>

削除処理に `GET` を使うと、以下のリスクが発生します：

- URLにアクセスするだけで削除されてしまう

- ブラウザのプレビューやリンク共有から誤って実行される可能性


そのため、**削除はPOSTで明示的に実行する設計に**しています。

<br>

### 3. ✅ **ユーザー権限のチェック**

```php
if ($_SESSION['role'] !== 'admin' && $_SESSION['role'] !== 'editor') {
    echo "You do not have permission to delete books.";
    exit;
}

```


誤操作や悪意あるユーザーによる削除を防ぐため、**管理者または編集者のみが削除操作を実行可能**にしています。

<br>

### 4. 🔒 **SQLインジェクション対策**

```php

$sql = 'DELETE FROM books WHERE id = :id';
$stmt = $dbh->prepare($sql);
$stmt->bindValue(':id', $id, PDO::PARAM_INT);
$stmt->execute();

```


プリペアドステートメントを使って変数をバインドし、**SQLインジェクション攻撃を防止**しています。

<br>

### 5. 📦 **存在確認とトランザクション処理**

```php

$dbh->beginTransaction();

// レコードの存在確認
// …

// 削除
// …

$dbh->commit();


```


削除前に対象のレコードが存在するかをチェックし、**処理の整合性を保つためトランザクションで実行**しています。途中で例外が発生した場合は `rollback()` されます。

<br>

### 6. 🧼 **XSS対策（str2html関数の活用）**

```php

echo "An error occurred: " . str2html($e->getMessage());

```

<br>

ユーザーに表示するエラー内容や出力内容には、`str2html()` を通すことで、**HTMLのタグとして解釈されないようにサニタイズ（無害化）**しています。

```php

function str2html(string $arg_input): string {
    return htmlspecialchars($arg_input, ENT_QUOTES, 'UTF-8');
}


```

<br>

### 📝 まとめ

削除処理は一見シンプルな機能に見えますが、**実装を誤ると意図しないデータ損失や攻撃の対象になる危険性**があります。今回の実装では、以下のようなポイントを押さえることで、安全な設計を実現しています：

- POSTメソッドの強制

- CSRFトークン検証

- ユーザー権限のチェック

- SQLインジェクション対策

- トランザクションによる整合性確保

- 出力時のXSS対策

<br>

### 参考

delete.php
```php
<?php
session_start();
require_once __DIR__ . '/../shared/login_check.php'; // Login checker
require_once __DIR__ . '/../shared/functions.php'; // Contains functions like str2html() and db_open()

// Check if the request method is POST
if ($_SERVER['REQUEST_METHOD'] !== 'POST') {
    echo "Invalid request method.";
    exit;
}

// Validate CSRF token
if (empty($_POST['token']) || empty($_SESSION['token']) || !hash_equals($_SESSION['token'], $_POST['token'])) {
    echo "Invalid CSRF token.";
    exit;
}

// Validate and sanitize the 'id'
$id = filter_input(INPUT_POST, 'id', FILTER_VALIDATE_INT);
if ($id === false) {
    echo "Invalid ID.";
    exit;
}

// Check user role for permission
if ($_SESSION['role'] !== 'admin' && $_SESSION['role'] !== 'editor') {
    echo "You do not have permission to delete books.";
    exit;
}


try {
    // Database connection
    $dbh = db_open();

    // Start transaction: Auto-Commit is now disabled
    $dbh->beginTransaction();

    // Check if the record exists
    $sql = 'SELECT COUNT(*) FROM books WHERE id = :id';
    $stmt = $dbh->prepare($sql);
    $stmt->bindParam(':id', $id, PDO::PARAM_INT);
    $stmt->execute();
    $count = $stmt->fetchColumn();

    if ($count == 0) {
        echo "The specified book ID was not found.";
        // Rollback the transaction
        $dbh->rollBack();
        exit;
    }

    // Delete the record
    $sql = 'DELETE FROM books WHERE id = :id';
    $stmt = $dbh->prepare($sql);
    $stmt->bindValue(':id', $id, PDO::PARAM_INT);
    $stmt->execute();

    // Commit the transaction
    $dbh->commit();

    // Log success
    error_log("Delete successful for Book id: " . $_POST['id'] . " ,User ID: " . $_SESSION['username']);
    echo "Book information has been deleted.<br>";
    echo "<a href='index.php'>Return to the list</a>";
} catch (PDOException $e) {
    // Rollback in case of error
    if ($dbh->inTransaction()) {
        $dbh->rollBack();
    }
    echo "An error occurred: " . str2html($e->getMessage()) . "<br>";

    // Rollback on any exception
    if ($dbh->inTransaction()) { // Check if a transaction is active before rollback
        $dbh->rollBack();
    }
    error_log($e->getMessage());

    exit;
}
?>

```



functions.php

```php

<?php
/**
 * Sanitize input for HTML output
 *
 * @param string $arg_input The input string to sanitize
 * @return string The sanitized string
 */
function str2html(string $arg_input) :string {
    return htmlspecialchars($arg_input, ENT_QUOTES, 'UTF-8');
}

function db_open(){
    $dsn = "mysql:host=localhost;dbname=sample_db";
    $user = "phpuser";
    $password = "dummy";
    $options = [
        PDO::ATTR_ERRMODE            => PDO::ERRMODE_EXCEPTION,
        PDO::ATTR_EMULATE_PREPARES   => false,
        PDO::MYSQL_ATTR_MULTI_STATEMENTS => false,
    ];
    $dbh = new PDO($dsn, $user, $password, $options);
    return $dbh;
}

```



