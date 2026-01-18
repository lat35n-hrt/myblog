+++
date = '2026-01-18T14:55:32+09:00'
draft = false
title = 'REST Assured PoC'
categories = ["REST Assured"]
+++

# macOSで最小構成のAPIテスト環境を作る（REST Assured + Java 21 + mvnw）

## この記事のゴール

REST Assured を使った API テストの **最小PoC** を作り、基本的な使い方を理解した状態にします。

今回の完成形は以下です。

- Java 21（Temurin）で動く
- Maven Wrapper(mvnw)で動く（macOSにMavenを深く入れない）
- REST Assured の基本形 `given` / `when` / `then` が使える
- 外部APIに依存せず、ローカルスタブで安定してテストが通る

## 前提条件（Versions / Environment）

- **OS**: macOS 13（Intel）
- **Java**: Temurin 21（OpenJDK 21.0.9 LTS）
- **Build**: Maven Wrapper（`./mvnw`）
  - Maven distribution: Apache Maven 3.9.12（`maven-wrapper.properties` で固定）
- **Test Framework**: JUnit 5.10.2
- **API Test DSL**: REST Assured 5.5.1
- **Local Stub Server**: OkHttp MockWebServer 4.12.0
- **Test Runner**: Maven Surefire Plugin 3.2.5
- **Compiler Target**: Java 21（`maven.compiler.source/target = 21`）

### 依存関係（pom.xml 由来のバージョン一覧）

- `org.junit.jupiter:junit-jupiter` 5.10.2
- `io.rest-assured:rest-assured` 5.5.1
- `io.rest-assured:json-path` 5.5.1
- `com.squareup.okhttp3:mockwebserver` 4.12.0
- `maven-surefire-plugin` 3.2.5

---

## 1. Java 21（Temurin）をHomebrewで入れる

まずは Java 実行環境のみ用意します。Homebrew で Temurin 21 を入れます。

```bash
brew install --cask temurin@21
```

確認：

```bash
java -version
/usr/libexec/java_home -V
```

`openjdk version "21.x"` が出ればOKです。

---

## 2. プロジェクト作成：rest_assured_poc

今回のPoCは以下のディレクトリをプロジェクトルートとします。

```bash
mkdir -p rest_assured_poc
cd rest_assured_poc
```

---

## 3. MavenをmacOSに入れずに、Maven Wrapper（mvnw）で回す

macOSにMavenをインストールしようとすると、環境によっては依存ライブラリの `make check` で止まることがあります。

PoC目的なら、ここに時間を使うメリットは薄いため Maven Wrapper（mvnw）を採用します。

### Maven Wrapper（only-script）を導入

`mvnw` をプロジェクト内に置き、Mavenはユーザー領域に落として使う方式です。

```bash
curl -L \
  https://repo.maven.apache.org/maven2/org/apache/maven/wrapper/maven-wrapper-distribution/3.3.4/maven-wrapper-distribution-3.3.4-only-script.zip \
  -o maven-wrapper.zip

unzip -q maven-wrapper.zip -x "mvnwDebug*"
rm -f maven-wrapper.zip
```

### Mavenバージョン固定（例：3.9.12）

```bash
mkdir -p .mvn/wrapper

cat > .mvn/wrapper/maven-wrapper.properties <<'EOF'
distributionUrl=https://repo.maven.apache.org/maven2/org/apache/maven/apache-maven/3.9.12/apache-maven-3.9.12-bin.zip
EOF
```

### 実行権限を付与：

```bash
chmod +x mvnw
./mvnw -v
```

ここで Maven バージョンが出れば、準備完了です。

---

## 4. pom.xml：REST Assured + JUnit5 + MockWebServer（最小）

Javaは依存管理を `pom.xml` に書くため、少し冗長に見えますが「こういうもの」です。

今回のPoCは、外部APIを叩かずに安定実行するため **MockWebServer** も追加します。

### pom.xml

```xml
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>poc</groupId>
  <artifactId>rest-assured-poc</artifactId>
  <version>0.1.0</version>

  <properties>
    <maven.compiler.source>21</maven.compiler.source>
    <maven.compiler.target>21</maven.compiler.target>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>

    <junit.version>5.10.2</junit.version>
    <restassured.version>5.5.1</restassured.version>
    <mockwebserver.version>4.12.0</mockwebserver.version>
  </properties>

  <dependencies>
    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>rest-assured</artifactId>
      <version>${restassured.version}</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>io.rest-assured</groupId>
      <artifactId>json-path</artifactId>
      <version>${restassured.version}</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>org.junit.jupiter</groupId>
      <artifactId>junit-jupiter</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>

    <dependency>
      <groupId>com.squareup.okhttp3</groupId>
      <artifactId>mockwebserver</artifactId>
      <version>${mockwebserver.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <artifactId>maven-surefire-plugin</artifactId>
        <version>3.2.5</version>
        <configuration>
          <useModulePath>false</useModulePath>
        </configuration>
      </plugin>
    </plugins>
  </build>
</project>
```

---

## 5. 最小のテスト：GET/POST（ローカルスタブ）

今回のPoCでは、Public API（例：reqres.in）を叩いてみたところ、環境によって 403で弾かれるケースがありました。

外部要因で落ちる設計は避けたいため、**MockWebServer** で擬似APIを立てて、テストを完全に再現可能にする構成に切り替えます。

### ディレクトリ

```bash
mkdir -p src/test/java/poc
```

### ApiSmokeTest.java

```java
package poc;

import io.restassured.RestAssured;
import okhttp3.mockwebserver.MockResponse;
import okhttp3.mockwebserver.MockWebServer;
import okhttp3.mockwebserver.RecordedRequest;
import org.junit.jupiter.api.*;

import java.io.IOException;

import static io.restassured.RestAssured.given;
import static org.hamcrest.Matchers.*;

public class ApiSmokeTest {

    private static MockWebServer server;

    @BeforeAll
    static void setup() throws IOException {
        server = new MockWebServer();
        server.start();

        // http://127.0.0.1:XXXXX/ を baseURI にする
        RestAssured.baseURI = server.url("/").toString();
    }

    @AfterAll
    static void teardown() throws IOException {
        server.shutdown();
    }

    @Test
    void get_users_should_return_page_2() {
        // Arrange: GET /users?page=2 のスタブ応答
        server.enqueue(new MockResponse()
                .setResponseCode(200)
                .addHeader("Content-Type", "application/json")
                .setBody("""
                {
                  "page": 2,
                  "data": [
                    {"id": 7, "email": "test@example.com"}
                  ]
                }
                """));

        // Act + Assert（REST Assured）
        given()
            .queryParam("page", 2)
        .when()
            .get("/users")
        .then()
            .statusCode(200)
            .body("page", equalTo(2))
            .body("data", not(empty()))
            .body("data[0].id", equalTo(7));

        // Optional: 実際に送ったリクエストも検証できる
        try {
            RecordedRequest req = server.takeRequest();
            Assertions.assertEquals("/users?page=2", req.getPath());
            Assertions.assertEquals("GET", req.getMethod());
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }

    @Test
    void create_user_should_return_201_and_id_createdAt() {
        // Arrange: POST /users のスタブ応答
        server.enqueue(new MockResponse()
                .setResponseCode(201)
                .addHeader("Content-Type", "application/json")
                .setBody("""
                {
                  "name": "michael",
                  "job": "qa",
                  "id": "123",
                  "createdAt": "2026-01-17T18:00:00Z"
                }
                """));

        String payload = """
        {
          "name": "michael",
          "job": "qa"
        }
        """;

        // Act + Assert（REST Assured）
        given()
            .contentType("application/json")
            .body(payload)
        .when()
            .post("/users")
        .then()
            .statusCode(201)
            .body("name", equalTo("michael"))
            .body("job", equalTo("qa"))
            .body("id", notNullValue())
            .body("createdAt", notNullValue());

        // Optional: 送信bodyの検証
        try {
            RecordedRequest req = server.takeRequest();
            Assertions.assertEquals("/users", req.getPath());
            Assertions.assertEquals("POST", req.getMethod());
            Assertions.assertTrue(req.getBody().readUtf8().contains("\"name\": \"michael\""));
        } catch (InterruptedException e) {
            throw new RuntimeException(e);
        }
    }
}
```

---

## 6. 実行：./mvnw test

Maven Wrapper で実行します。

```bash
./mvnw test
```

成功すると以下のように出ます。

```
Tests run: 2, Failures: 0

BUILD SUCCESS
```



![API Smoke Test Result](api_smoke_test_result.png)

---

## 7. このPoCで理解できたポイント

この最小PoCで、REST Assured について以下が確認できました。

- `given`/`when`/`then` で API テストを記述できる
- ステータスコードと JSON ボディの検証（Hamcrest）が可能
- 外部APIは 403 / rate limit などで不安定になり得る
  → **MockWebServer** でローカルスタブにして再現性を担保できる
- 「仕様どおりのリクエストを投げているか」も `RecordedRequest` で検証できる

外部API依存だと環境要因でテストが落ちる可能性があるため、最小PoCではローカルスタブに寄せて  `deterministic（決定論的）` な実行環境を作ることができました。

---

## まとめ

REST Assured の基本を理解するための最小PoCとして、以下を構築しました。

- Java 21（Temurin）
- Maven Wrapper（mvnw）
- REST Assured の smoke テスト2本（GET/POST）
- 外部依存を排除したローカルスタブ（MockWebServer）

これで REST Assured の基本的な使い方と、安定したテスト環境の作り方が理解できました。
