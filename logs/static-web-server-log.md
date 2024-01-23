# 作業ログ: Static Web Server

2024.1.23

目標：接続方法などを思い出す＆ドメイン取得までやる

本番環境の構築

- ec2インスタンスへのssh接続… `ssh -i /path/StaticWebServer.pem ubuntu@54.252.135.3`
- (`ssh -i /path/key-pair-name.pem instance-user-name@public-ip-address`)
- ipアドレスとかはEC2の詳細から確認できる（毎回変わってるかもしれない）

本番環境でのgitお試しpushは済ませてあることを確認した

nginxインストールされてることも確認(nginx -v)

nginx起動確認(`sudo systemctl start nginx`)

→起動できなかった

`sudo systemctl start nginx`

`Job for nginx.service failed because the control process exited with error code.`

`See "systemctl status nginx.service" and "journalctl -xeu nginx.service" for details.`

statusを確認

`sudo systemctl status nginx.service`

```
 Loaded: loaded (/lib/systemd/system/nginx.service; enabled; vendor preset: enabled)

 Active: failed (Result: exit-code) since Tue 2024-01-23 04:49:08 UTC; 2min 31s ago
// ↑作業再開した日からinactiveになってる

   Docs: man:nginx(8)

Process: 144417 ExecStartPre=/usr/sbin/nginx -t -q -g daemon on; master_process on; (code=exited, status=1/FAILURE)

    CPU: 12ms

```

Jan 23 04:49:07 ip-172-31-12-208 systemd[1]: Starting A high performance web server and a reverse proxy server...

Jan 23 04:49:08 ip-172-31-12-208 nginx[144417]: nginx: [emerg] "location" directive is not allowed here in /etc/nginx/nginx.conf:85

Jan 23 04:49:08 ip-172-31-12-208 nginx[144417]: nginx: configuration file /etc/nginx/nginx.conf test failed

Jan 23 04:49:08 ip-172-31-12-208 systemd[1]: nginx.service: Control process exited, code=exited, status=1/FAILURE

Jan 23 04:49:08 ip-172-31-12-208 systemd[1]: nginx.service: Failed with result 'exit-code'.

Jan 23 04:49:08 ip-172-31-12-208 systemd[1]: Failed to start A high performance web server and a reverse proxy server.

エラーログ確認

`sudo nginx -t` 

nginx: [emerg] "location" directive is not allowed here in /etc/nginx/nginx.conf:85

nginx: configuration file /etc/nginx/nginx.conf test failed

serverのブロックに囲まれてないlocationディレクティブによってシンタクスエラーが起きてた

→とりあえずコメントアウトしといて解消

start後、パブリックipアドレスをアドレスバーに入力すればアクセスページが表示される

nginxのデフォルト設定が済ませてあったので、プロジェクトのhtmlページが表示された

ドメイン名の取得

- namecheapから取得 (kano.wiki)

DNS設定

- [namecheapのdns設定方法](https://www.namecheap.com/support/knowledgebase/article.aspx/434/2237/how-do-i-set-up-host-records-for-a-domain/)
- www.deepsea@kano.wiki

独自ドメインでdeepsea.htmlにアクセスできた

2024.1.24

目標：サブドメインごとに違うコンテンツを表示させる、HTTPSに設定する

サーバーブロック

- /etc/nginx/sites-available/deepsea.{your_domain}にdeepseaサイト用の設定ファイルを作成

`server {
    listen 80;
    listen [::]:80;

    root /var/www/project-deepsea/public/project-deepsea-website;
    index deepsea.html;

    server_name deepsea.{your_domain};

    location / {
        try_files $uri $uri/ =404;
    }
}`

- 設定ファイルへのシンボリックリンク作成　`sudo ln -s /etc/nginx/sites-available/deepsea.{your_domain} /etc/nginx/sites-enabled/`

→kano.wikiでNginxのアクセスページ、deepsea.kano.wikiでdeepsea.htmlが表示された

RSA暗号化

- サーバーは２つの大きな素数（秘密鍵）を生成する（サーバーだけが知る）
- →クライアントとサーバーで素数の積nを共有する　n = p * q
- →クライアントは秘密鍵xを生成
- →サーバーにメッセージを送るときに、クライアントは公開鍵n, sで暗号化する（x^s%n = 暗号化メッセージ）
- ↑の余りの数、秘密鍵p,q、公開鍵nを元に、サーバーはクライアントが使った秘密鍵xを復元
- ＊復元に必要なp, qはサーバーしか知らないため、第三者は解読できないということになる

HTTPS

- 安全な接続のためには、セッションキーが必要
- 一時的な接続に必要なセッションキーをクライアントが生成し、サーバー側から取得できるように取得するための情報をRSA暗号化して送信
- TLSという暗号化技術のもとに成り立つ

TLS（トランスポートレイヤーセキュリティ）

- メッセージは公開鍵で暗号化、秘密鍵を持つ人のみ解読可能
- TLS証明書による認証が必要
- MAC（メッセージ認証コード）を送信者のものと照合してデータチェック

- ハンドシェイク（通信方法の確認や両者の認証）
- →クライアント・サーバはそれぞれ乱数を生成
- →使用するTLSのバージョン決定
- →暗号スイートの選択
- →公開鍵の信頼性証明のため、サーバがTLS証明書を作成
- →サーバの公開鍵をもとに、クライアントがプレマスター秘密鍵を生成
- →さらにサーバに送られ、プレマスター秘密鍵、乱数、サーバ自身の秘密鍵からサーバはマスター秘密鍵を作成
- →今後のメッセージの暗号化・復号化に利用される

NginxでのHTTP設定

- パッケージマネージャーのsnap coreインストール
- ↑を使ってcertbotインストール
- binディレクトリにcertbotへのシンボリックリンク作成　`sudo sudo ln -s /snap/bin/certbot /usr/bin/certbot`

- `sudo certbot --nginx -d deepsea.{your_domein}.com`
- →画面の案内にしたがっていくと証明書と秘密鍵が生成される
- httpsになった

`sudo certbot renew --dry-run`

Saving debug log to /var/log/letsencrypt/letsencrypt.log

---

Processing /etc/letsencrypt/renewal/deepsea.kano.wiki.conf

---

Account registered.

Simulating renewal of an existing certificate for deepsea.kano.wiki

---

Congratulations, all simulated renewals succeeded:

/etc/letsencrypt/live/deepsea.kano.wiki/fullchain.pem (success)

↑証明書の自動更新スクリプトは機能している
