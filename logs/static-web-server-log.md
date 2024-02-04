# 作業ログ：Static Web Server（Resume Site）

[参考記事](https://choippo.com/nextjs-pm2/)

2024.1.23

目標：接続方法などを思い出す＆ドメイン取得までやる

[EC2インスタンスセットアップ手順](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/EC2_GetStarted.html)

- ローカルの~/.sshディレクトリに.pemファイルを保存
- 権限渡す　`chmod 400 StaticWebServer-Pair.pem`
- ↓SSH接続

本番環境の構築

- ec2インスタンスへのssh接続… `ssh -i ~/.ssh/StaticWebServer-Pair.pem [ubuntu@13.239.26.30](mailto:ubuntu@13.239.26.30)`
- (`ssh -i /path/key-pair-name.pem instance-user-name@public-ip-address`)
- ipアドレスとかはEC2の詳細から確認できる

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
- binディレクトリにcertbotへのシンボリックリンク作成　`sudo ln -s /snap/bin/certbot /usr/bin/certbot`

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

---

1/25

Resume Site（仮）デプロイ

デプロイ方法の復習として、仮のportfolioプロジェクトデプロイ手順をもう一回やる

- sudo mkdir -p /var/www/project-protfolio/public
- 現在のユーザーに権限渡す　sudo chown -R $USER:USER /var/www/project-portfolio/public
- シンボリックリンク作成　`sudo ln -s ~/web/portfolio /var/www/project-portfolio/public`
- nginxアクセス権限設定　chmod 755 /home/ubuntu
- シンボリックリンク確認　ls -l /var/www/project-portfolio/public と ls -l /var/www/project-portfolio/public/portfolio/
- ポートフォリオサイト用の新しいリポジトリを作って、portfolio.kano.wiki用の設定ファイルを/etc/nginx/sites-available/に作成する

前回と同じく静的ページのみ公開でもよかったけど、Next.jsプロジェクトの公開を試してみる

- 本番環境にリポジトリをpull後、`npm run build`でデプロイ
- pm2でアプリを起動させて永続化　`pm2 start npm —name “portfolio” — start`

[Next.jsプロジェクトをNginxで公開（参考記事）](https://dev.to/j3rry320/deploy-your-nextjs-app-like-a-pro-a-step-by-step-guide-using-nginx-pm2-certbot-and-git-on-your-linux-server-3286)

`/etc/nginx/sites-available/portfolio`

```
server {
        server_name portfolio.kano.wiki www.portfolio.kano.wiki;

        location / {
                proxy_pass http://localhost:3000;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/portfolio.kano.wiki/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/portfolio.kano.wiki/privkey.pem; # managed by Cert>
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = portfolio.kano.wiki) {
        return 301 https://$host$request_uri;
    } # managed by Certbot

        listen 80;
        server_name portfolio.kano.wiki www.portfolio.kano.wiki;
    return 404; # managed by Certbot

}

```

様々な記事で、pm2によるNodeサーバーの永続化（デーモン化）について触れられていた。後々深く調べる。

[解説してる記事](https://choippo.com/nextjs-pm2/)

create next app直後の段階で一旦公開テスト

表示できた

[https://portfolio.kano.wiki/](https://portfolio.kano.wiki/)

---

1/30

しばらくEC2インスタンスを停止していた状態から作業再開

*パブリックIPアドレスは前回と別のものに割り当てられているので、dnsとfilezillaの設定からポート番号を新しく変える必要がある

目標：ポートフォリオをWorksページに掲載、変更後に手動で再デプロイ

- portfolio.jsonから制作物一覧を取得してカードを表示

`public/portfolio.json`

```json
[
  {
    "id": 1,
    "title": "Burger Shop Game / バーガーショップゲーム",
    "summary": "バーガーショップでの販売を体験できるクリッカーゲームです。",
    "content": "Burger Shop Gameは、バーガーショップでハンバーガーを販売してお金を稼ぐことでバーガーショップ店員の体験ができる、暇つぶしにちょうどいいクリッカーゲームです。",
    "url": "https://southernminami.github.io/BurgerShopGame/src/index.html",
    "github": "https://github.com/SouthernMinami/BurgerShopGame",
    "thumbnail": "/assets/burger-shop-game.png",
    "date": "2024-01-27T00:00:00"
  },
  {
    "id": 2,
    "title": "Playing Card / トランプゲーム",
    "summary": "4つのゲームモードで遊べるシンプルなトランプゲームです",
    "content": "Playing Cardは、4つのトランプゲームで遊べるシンプルなトランプゲームです。\n\nこのツールの開発に当たっては、バックエンドとフロントエンドの両方を私一人で担当しました。バックエンドはNode.jsとExpressを使用し、フロントエンドはReact.jsで構築しました。データベースにはMongoDBを用い、ユーザーデータと金融データの管理を行いました。\n\nBudgetMasterはウェブ上で公開され、多くのユーザーから肯定的なフィードバックを得ています。特に、インターフェースの使いやすさと、予算追跡の明確さがユーザーから好評を博しています。このプロジェクトを通じて、私はデータ駆動型のユーザーインターフェースの設計と、セキュアなユーザーデータの管理について深く学ぶことができました。",
    "url": "https://github.com/",
    "github": "https://github.com/SouthernMinami/BurgerShopGame",
    "thumbnail": "/assets/playing-card.png",
    "date": "2024-01-27T00:00:00"
  },
  {
    "id": 3,
    "title": "Markdown Converter / mdファイル変換スクリプト",
    "summary": "mdファイルをHTMLファイルに変換するPythonスクリプトです。",
    "content": "Markdown Converterは、mdファイルをHTMLファイルに変換するPythonスクリプトです。このスクリプトは、mdファイルをHTMLファイルに変換するだけでなく、mdファイルの内容を解析して、HTMLファイルに適切なタイトル、サブタイトル、段落、リスト、画像、リンクを挿入します。\n\nこのスクリプトの開発には、Pythonを使用してmdファイルを解析し、HTMLファイルに変換する方法を学びました。また、このプロジェクトを通じて、私はPythonの基本的な構文と、ファイルの読み書きについて学ぶことができました。",
    "url": "https://github.com/SouthernMinami/MarkdownConverter",
    "github": "https://github.com/SouthernMinami/MarkdownConverter",
    "thumbnail": "/assets/markdown-converter.png",
    "date": "2024-01-27T00:00:00"
  },
  {
    "id": 4,
    "title": "Online Chat Messenger / ターミナル上のチャットアプリ",
    "summary": "ターミナル上で動作するチャットアプリです。",
    "content": "Online Chat Messengerは、ターミナル上で動作するチャットアプリです。このアプリは、サーバーとクライアントの2つのプログラムで構成されています。サーバーは、クライアントからの接続を待ち受け、クライアントはサーバーに接続してメッセージを送受信します。\n\nこのアプリの開発には、Pythonのソケットプログラミングを使用しました。このプロジェクトを通じて、私はPythonのソケットプログラミングについて学ぶことができました。",
    "url": "https://github.com/Recursion-BackendNovice-TeamA/TeamDev-OnlineChatMessenger",
    "github": "https://github.com/Recursion-BackendNovice-TeamA/TeamDev-OnlineChatMessenger",
    "thumbnail": "/assets/online-chat-messenger.png",
    "date": "2024-01-27T00:00:00"
  },
  {
    "id": 5,
    "title": "Audio Visualizer / オーディオビジュアライザー",
    "summary": "TouchDesignerを使用して作成したオーディオビジュアライザーです。",
    "content": "Audio Visualizerは、TouchDesignerを使用して作成したオーディオビジュアライザーです。このビジュアライザーは、音楽ファイルを読み込んで、音楽の波形を可視化します。\n\nこのプロジェクトを通じて、私はTouchDesignerの基本的な機能と、音楽の波形を可視化する方法を学ぶことができました。",
    "url": "",
    "github": "",
    "thumbnail": "/assets/audio-visualizer.png",
    "date": "2023-05-30T00:00:00"
  }
]
```

`works/page.jsx`

```tsx
'use client'

import { useEffect, useState } from 'react'
import { Card } from '../components/elements/Card'

const Works = () => {
  const [works, setWorks] = useState([])

  useEffect(() => {
    const xhr = new XMLHttpRequest()

    xhr.open('GET', 'portfolio.json')
    xhr.onload = () => {
      if (xhr.status === 200) {
        setWorks(JSON.parse(xhr.response))
      } else {
        console.error('データの取得に失敗しました。')
      }
    }
    xhr.send()
  }, [])

  return (
    <div>
      <div className="flex flex-col items-center justify-center p-4">
        <h1 className="text-3xl">Works</h1>
      </div>
      <ul className="flex flex-wrap justify-center">
        {works.map((work, index) => {
          return <Card key={index} work={work} />
        })}
      </ul>
    </div>
  )
}

export default Works
```

- サイトの更新は自動化したりしてないので、push後また手動でbuild

`git pull`

`npm run build`

[https://portfolio.kano.wiki/works](https://portfolio.kano.wiki/works)

pm2で起動したアプリも再起動

`pm2 restart portfolio`

---

1/31

目標：要件最低限満たすとこまでやる（やれそう）

**機能要件**
• [x]  サブドメイン(portfolio.kano.wiki)からアクセス可能である
• [x]  ホームページに自身の紹介と他のページの説明がある
• [x]  これまでに作った全てのポートフォリオを掲載するWorksページがあ
• [x]  専門職に関連したスキルや経験を表示するResumeページがある
• [ ]  履歴書の PDF 版をダウンロードするためのリソースファイルが提供される
• [x]  サイトは HTTPS でアクセス可能である
• [x]  全てのページは、リンクとフッターを含むナビゲーションを備えた統一したレイアウトデザインに従っている
• [x]  ウェブサーバは、パス URL スキームに基づいて全ての公開リソース（（ウェブページ、画像、動画、スタイルシート、スクリプトなど）を提供する
**非機能要件**
• [x]  サーバのセキュリティが確保されている。これには、ユーザーのセキュリティと、サーバにアクセスする他の人々からのセキュリティが含まれます。
• [x]  サーバの稼働率は、年間を通じて 99% 以上を維持する
• [x]  開発者が容易に開発でき、ローカル環境から本番環境へのアップデートを簡単にプッシュできる。
• [x]  開発者は、単一のコマンドを実行するだけでローカルのコードをサーバに同期させることができる(npm run build)
• [x]  新しいインスタンスを作成してプロジェクトを行う。(ウェブサーバのセットアップとデプロイの経験に慣れるため）CancelUpdate comment

- 履歴書PDFダウンロード以外の全ページのコンテンツを追加
- ページトップにスクロールさせるボタン追加

[サイトURL](https://portfolio.kano.wiki)

履歴書PDFの追加にすぐにとりかかる

*npm run build時に、ビルドが終わらない問題発生

原因：EC2のCPU使用率がほぼ１００％になってた

→インスタンス再起動で解決（また各種設定のIPアドレスを変えなきゃ）

CPU使用率が急激に上がってた原因は不明（Next.jsかpm2が関係ある？）

---

# 2/1

目標：PDFダウンロード機能追加、スマホとPCで文字の色同じにする、アイコン画像作成

- global.cssのbody color: blackに変えてスマホもPCもデフォルトは黒文字になった
- `<a “href=”path/to/file” download>` ダウンロードボタン
- Gimpでアイコン作った　（Ctrl + Shift押しながらで書き始めからの垂直線が引ける）

要件満たしたので、いったん区切り

他のバックエンドPJ終わり次第、レイアウト凝る
