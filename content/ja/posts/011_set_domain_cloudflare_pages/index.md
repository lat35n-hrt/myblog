+++
date = '2025-05-07T19:24:37+09:00'
draft = false
title = 'Cloudflareに独自ドメイン適用'
categories = ["Cloudflare"]
+++



## 目的

ブログに独自ドメインを適用する


## 状況整理
・ドメイン取得元：さくらインターネット（＝レジストラ）

・ブログ運用先：Cloudflare（＝DNS管理＆CDN/セキュリティ/ホスティング）



## 手順（概要）

1. Cloudflare にドメインを追加

・Cloudflare にログイン

・「Free」プランを選択して進む（有料プランは不要）

・「Add Domain」で新しく取得したドメインを Site に追加
(追加したドメイン名の状態はまだ Invalid nameservers )

・ DNSレコードをセットアップ（必要に応じてAレコード、CNAMEなど。今回は自動でした）

2. Cloudflare が指示するネームサーバーを確認

例: clark.ns.cloudflare.com, dina.ns.cloudflare.com

3. さくらインターネットの管理画面にログイン

・ ドメイン設定から既存の「ネームサーバー」を削除

・ Cloudflare指定のネームサーバーを追加

4. 反映を待つ

完了後、Cloudflare のダッシュボード上で「Status: Active」と表示されれば完了！


✅ 設定ポイント

・ DNSレコードの設定
Cloudflareダッシュボードの「DNS」タブでCNAMEを確認：

| Type  | Name        | Content                |
|-------|-------------|------------------------|
| CNAME | yourdomain.com | yourblog.pages.dev   |


これにより、ドメイン名 (yourdomain.com) を入力すると、
Cloudflare Pages にホスティングされているコンテンツ（yourblog.pages.dev）が表示される設定となります。


---
💡  Aレコードがない場合について

Aレコードは、ドメインを 直接IPアドレス に解決させるものです。
例えば、Webホスティングや自前のサーバーを使っている場合は、AレコードでそのサーバーのIPを指定します。

Cloudflare Pages のように、静的ホスティングサービスを使っている場合は、通常、CNAMEレコードだけで十分です。

したがって、今回のように CNAMEが設定されていれば、Aレコードを追加する必要はありません。

---


・ SSL/TLSの設定
Cloudflare の「SSL/TLS」設定を Full または Full (Strict) にしておくと、HTTPSが自動的に使えます。

・ キャッシュや最適化の確認
必要に応じて、「Speed」や「Caching」タブで、ページキャッシュや圧縮の最適化を設定できます。


5. ブラウザからドメイン名にアクセスして動作を確認
<p></p>


6. config.toml の baseURL を変更（本来は最初にやっておく）

・baseURL = "https://yourdomain.com/" を変更

・hugo コマンドで再ビルド

・GitHub に push して Cloudflare Pages に再デプロイ


