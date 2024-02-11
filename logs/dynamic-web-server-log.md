# 作業ログ：Dynamic Web Server

# 2/1

目標：PHP周りの環境構築終了

新しい本番環境用インスタンス作成　[手順](https://www.notion.so/Static-Web-Server-Resume-Site-c26e6560e0e24bdda640fee29e0bb4bb?pvs=21)

chmod 400 ???.pem

ローカル・本番環境両方でphpインストール

Ubuntu(EC2): `sudo apt update`    `sudo apt install php`

Windows(ローカル): [https://www.javadrive.jp/php/install/index1.html](https://www.javadrive.jp/php/install/index1.html)

(* Windowsにインストールはかなり詰まったのでローカルもUbuntu（VM）でやる）

PHPの慣れてない記法

- `isset($variable)` … 変数が定義されてるかどうかを返す（1 or 何も表示しない）
- 定数には$はつかない　`const CONSTANT = “”;`
- `strpos()` … 文字列内で部分文字列が最初に出てくる場所を返す
- `mixed` … 型チェックがない複数のデータ型　anyみたい
- phpの配列はメソッドを持つオブジェクトではないので、配列の長さ取得は`count($arr)`で行う
- 連想配列の書きかた … `[’a’ ⇒ 0, ‘b’ ⇒ 1, …]`
- オブジェクトのメンバー変数・メソッドへのアクセスはobj.valではなく`obj⇒$val`
- ラムダ関数

```php
<?php
// 関数はcallable型
function processNumbers(array $numbers, callable $callback): array {
    // array_mapは配列の各要素に関数を適用します
    return array_map($callback, $numbers);
}

function squareNumber(int $number): int {
    return $number * $number;
}

$nums = [0,1,2,3];
// 関数は'関数名'で渡す
$squaredNumbers = processNumbers($numbers, 'squareNumber');
print_r($squaredNumbers);
```

- nullable… nullを持つ型(`mix`と`?`が該当）

```php
<?php
function findValue(array $data, string $key): mixed {
    // ?? 演算子は左のパラメータを返し、それがない場合はnullを返します
    return $data[$key] ?? null;
}

// 使用例:
$data = ['name' => 'John', 'age' => 30];
echo findValue($data, 'name'); // Output: John
echo findValue($data, 'occupation'); // Output: (何もなし、'occupation'というキーは存在しないため)
```

- `?type` … null || type (maybe{type})

```php
<?php
function calculateDiscount(?string $subscriptionLevel): maybe{int} {
    $discounts = [
        'silver' => 10,
        'gold' => 20,
        'platinum' => 30,
    ];

    // $discounts配列内にサブスクリプションレベルが存在するか確認します
    if ($subscriptionLevel !== null && isset($discounts[$subscriptionLevel])) {
        return $discounts[$subscriptionLevel];
    } else {
        // サブスクリプションレベルが認識されない場合はデフォルトの割引値を返します
        return 5;
    }
}
```

クラスの組織構造

- `namespace` … 名前空間の概念に基づいて、関連するクラス群を１つのファイルパスに整理するときに使う（他の変数を持つ変数を表す方の名前空間の意味ではない）

インターフェース用ディレクトリ

Interfaces/Engine.php (

```php
namespace Interfaces;

interface Engine {
    public function start(): string;
}
```

Engineインターフェースを実装しているクラス用のディレクトリ

Engines/GasolineEngine.php

```php
namespace Engines;

use Interfaces\Engine;

class GasolineEngine implements Engine {
    public function start(): string {
        return "Starting the gasoline engine...";
    }
}
```

Engines/ElectricEngine.php

```php
namespace Engines;

use Interfaces\Engine;

class ElectricEngine implements Engine {
    public function start(): string {
        return "Starting the electric engine...";
    }
}
```

Car抽象クラスを含むCar関連クラス用ディレクトリ

Cars/Car.php

```php
namespace Cars;

use Interfaces\Engine;

abstract class Car {
    protected string $make;
    protected Engine $engine;

    public function __construct(string $make, Engine $engine) {
        $this->make = $make;
        $this->engine = $engine;
    }

    abstract public function drive(): string;

    public function start(): string {
        return $this->engine->start();
    }
}
```

Cars/GasCar.php

```php
namespace Cars;

use Engines\GasolineEngine;

class GasCar extends Car {
    public function __construct(string $make) {
        parent::__construct($make, new GasolineEngine());
    }

    public function drive(): string {
        return "Driving the gas car...";
    }
}
```

Cars/ElectricCar.php

```php
namespace Cars;

use Engines\ElectricEngine;

class ElectricCar extends Car {
    public function __construct(string $make) {
        parent::__construct($make, new ElectricEngine());
    }

    public function drive(): string {
        return "Driving the electric car...";
    }
}
```

エントリポイント

- `include` ... 指定したファイルが見つからなくてもスクリプトを続行する
- `require` … 指定したファイルが見つからなかったらスクリプトの実行を中断する
- `include_once, requre_once` … 読み込み済みのファイルだった場合は再度読み込みをしない
- slp_autoload_register()  … 未定義クラス・インターフェースが読み込まれるとき、それらを定義するためのコードを自動で読み込む

```php
spl_autoload_extensions(".php"); 
spl_autoload_register();

// autoloaderのおかげで、手動インポートなしでクラスを読み込める
$gasCar = new Cars\GasCar('Toyota');
$electricCar = new Cars\ElectricCar('Tesla');

echo $gasCar->drive(); // Output: Driving the gas car...
echo $gasCar->start(); // Output: Starting the gasoline engine...

echo $electricCar->drive(); // Output: Driving the electric car...
echo $electricCar->start(); // Output: Starting the electric engine...
```

引数を渡すこともできる

```php
spl_autoload_register( function($name) {
    // __DIR__は、現在のファイルの絶対ディレクトリパスを取得します。
    $filepath = __DIR__ . "/" . str_replace('\\', '/', $name) . ".php";
    echo "\nRequiring...." . $name . " once ($filepath).\n";
    // バックスラッシュ(\)をフロントスラッシュ(/)に置き換えます。フロントスラッシュはLinuxのファイルパスで使用されます。
    require_once $filepath;
});

$gasCar = new Cars\GasCar('Toyota');
$electricCar = new Cars\ElectricCar('Tesla');

echo $gasCar->drive(); // Output: Driving the gas car...
echo $gasCar->start(); // Output: Starting the gasoline engine...

echo $electricCar->drive(); // Output: Driving the electric car...
echo $electricCar->start(); // Output: Starting the electric engine...
```

---

# 2/2

目標：Food Service Simulator作成

[https://github.com/SouthernMinami/FoodServiceSimulation](https://github.com/SouthernMinami/FoodServiceSimulation)

# 2/3

目標：アプリケーションサーバーとローカル動的サーバーの概要理解、Restaurant Chain Mockup作成

### アプリケーションサーバー（動的ウェブサーバー）

- PHPはHTTPリクエストに応答するアプリケーションサーバーの役割を果たす
- 静的ウェブサーバー…クライアントから要求された静的なファイルを直接送信　それに対して↓
- 動的ウェブサーバー…要求に応じてリアルタイムでコンテンツを生成しクライアントに送信（DBからのデータ取得など）

リバースプロキシ

- 参照　[SNSのウェブサーバ](https://recursionist.io/dashboard/course/33/lesson/1144)
- ウェブサーバを変換し、クライアントからのリクエストを別のサーバー（アプリケーションサーバー）に転送するための仲介役として扱う手法
- （これによって大規模なアプリに有効なロードバランシングを実装できる）
- サーバーとして扱えるNginxもリバースプロキシに変換できる

*Next.jsで作成したResumeサイトをデプロイする際も、リバースプロキシとしてのNginxがNode.jsのHTTPサーバーにリクエストを転送して処理してもらってるということになっているはず

要復習

PHP-FPM

- PHP用の実行環境
- 事前にプロセスを起動しておいて再利用することで、毎回インタプリタを起動せずに済む
- Nginx → PHP-FPMに特定のURLパターンのリクエストを転送できるように設定する

本番環境の前に、ローカルでPHPサーバーを動かしてみる

### ローカル動的サーバー

PHP組み込みサーバー

- デフォルトではindex.phpとURLをマッチングさせてある

```php
<!DOCTYPE html>

<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Hello, World</title>
</head>
<body>
    <h1>Hello, World</h1>
</body>
</html>
```

- `php -S [localhost:8000](http://localhost:8000)` サーバー起動
- `-S`はサーバーオプションで、localhostはループバックアドレス（127.0.0.1)に変換される
- `<?php statement ?>`でPHPの処理を埋め込める
- `%s`… String, `%d`… Decimal, `%f`… float, `%b`… binary, `%x`… Hexadecimal

```php
<?php $planet = 'Jupiter'; ?>
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Hello, World!</title>
</head>
<body>
<h1>Hello, <?php echo $planet;?></h1>
<p>This is a simple "Hello, <?php echo $planet;?>!" HTML page.</p>
<footer>
    <p><?php echo sprintf("%s Rocks©  Website %s. All rights reserved.", $planet, date("Y")); ?></p>
</footer>
</body>
</html>
```

HTTPデータ取得用のグローバル変数

| 変数名 | 内容 |
| --- | --- |
| $_GET | リクエストのクエリパラメータ（?parameter=value)のところ）を取得 |
| $_POST | HTTPリクエストがPOSTした際に、リクエストボディの内容を取得してデータを連想配列に変換 |
| $_REQUEST | POSTリクエストのボディ、GETのクエリパラメータ、COOKIEデータを一括で取得 |

```php
<?php $planet = isset($_GET['planet']) ? $_GET['planet'] : 'Earth' ?>
<!DOCTYPE html>
<html lang="en">
<head>
    <title>Hello, World!</title>
</head>
<body>
<h1>Hello, <?php echo $planet;?></h1>
<p>This is a simple "Hello, <?php echo $planet;?>!" HTML page.</p>

<footer>
    <p><?php echo sprintf("%s Rocks©  Website %s. All rights reserved.", $planet, date("Y")); ?></p>
</footer>
</body>
</html>
```

Dev Tool → Network → localhostからHTTPリクエスト内容が見れる（何もGETできなかったからHello, Earthが表示されてる）

コンテンツタイプの上書き

```php
<?php
// レスポンスヘッダーのContent-Typeを変える
header('Content-Type: application/json');
// get_headers() ... すべてのリクエストヘッダを連想配列で取得
// 取得に失敗したら{}の部分が返される
echo json_encode(get_headers() ?? '{}');
?>

			
```

JSONファイルをダウンロードする形式にできる

```php
<?php
header('Content-Type: application/json');

$filename = 'request_headers.json';
// Content-Disposition: attachment ... レスポンスをダウンロードするように指示 
header(sprintf('Content-Disposition: attachment; filename="%s"', $filename));

echo json_encode(getallheaders() ?? '{}');
?>
```

### Lorem Ipsum

ランダムなユーザージェネレーター

- /Models/Userクラス

```php
<?php

namespace Models;
use DateTime;

class User {
    private int $id;
    private string $firstName;
    private string $lastName;
    private string $email;
    private string $password;
    private bool $isActive;
    private string $hashedPassword;
    private string $phoneNumber;
    private string $address;
    private DateTime $birthDate;
    private DateTime $membershipExpirationDate;
    private string $role;

    public function __construct(int $id, string $firstName, string $lastName, string $hashedPassword, string $phoneNumber, string $address, DateTime $birthDate, DateTime $membershipExpirationDate, string $role) {
        $this->id = $id;
        $this->firstName = $firstName;
        $this->lastName = $lastName;
        $this->hashedPassword = $hashedPassword;
        $this->phoneNumber = $phoneNumber;
        $this->address = $address;
        $this->birthDate = $birthDate;
        $this->membershipExpirationDate = $membershipExpirationDate;
        $this->role = $role;
    }

    public function login(string $password): bool {
        // ハッシュ済みのパスワードを使って認証
        return password_verify($password, $this->hashedPassword);
    }

    public function updateProfile(string $address, string $phoneNumber): void {
        $this->address = $address;
        $this->phoneNumber = $phoneNumber;
    }

    public function changePassword(string $newPassword): void {
        $this->hashedPassword = password_hash($newPassword, PASSWORD_DEFAULT);
    }

    public function hasMembershipExpired(): bool {
        $currentDate = new DateTime();
        return $currentDate >= $this->membershipExpirationDate;
    }

    public function toString(): string {
        return sprintf(
            "User ID: %d\nName: %s %s\nEmail: %s\nPhone Number: %s\nAddress: %s\nBirth Date: %s\nMembership Expiration Date: %s\nRole: %s\n",
            $this->id,
            $this->firstName,
            $this->lastName,
            $this->email,
            $this->phoneNumber,
            $this->address,
            $this->birthDate->format('Y-m-d'),
            $this->membershipExpirationDate->format('Y-m-d'),
            $this->role
        );
    }

    public function toHTML() {
        return sprintf("
            <div class='user-card'>
                <div class='avatar'>SAMPLE</div>
                <h2>%s %s</h2>
                <p>%s</p>
                <p>%s</p>
                <p>%s</p>
                <p>Birth Date: %s</p>
                <p>Membership Expiration Date: %s</p>
                <p>Role: %s</p>
            </div>",
            $this->firstName,
            $this->lastName,
            $this->email,
            $this->phoneNumber,
            $this->address,
            $this->birthDate->format('Y-m-d'),
            $this->membershipExpirationDate->format('Y-m-d'),
            $this->role
        );
    }

    public function toMarkdown() {
        return "## User: {$this->firstName} {$this->lastName}
                 - Email: {$this->email}
                 - Phone Number: {$this->phoneNumber}
                 - Address: {$this->address}
                 - Birth Date: {$this->birthDate->format('Y-m-d')}
                 - Is Active: {$this->isActive}
                 - Role: {$this->role}";
    }

    public function toArray() {
        return [
            'id' => $this->id,
            'firstName' => $this->firstName,
            'lastName' => $this->lastName,
            'email' => $this->email,
            'password' => $this->password,
            'phoneNumber' => $this->phoneNumber,
            'address' => $this->address,
            'birthDate' => $this->birthDate,
            'isActive' => $this->isActive,
            'role' => $this->role
        ];
    }
}
```

- Composerを使ってfakerをインストール

`sudo apt update`

`sudo apt install php-cli phps-zip unzip` 依存関係インストール

`curl -sS https://getcomposer.org/installer -o composer-setup.php` Composerをダウンロード

`HASH="$(curl -sS [https://composer.github.io/installer.sig](https://composer.github.io/installer.sig))"`  `echo "$HASH composer-setup.php" | sha384sum -c -`  ダウンロードしたインストーラのSHA-384ハッシュを比較して、インストーラが正しいかどうか検証

`sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer`　インストーラを実行

composer.jsonを作成

```json
{
    "require": {
        "fakerphp/faker": "^1.15"
    }
}
```

composer require fakerphp/faker　fakerをインストール

vendorフォルダがインストールされ、`require_once ‘vendor/autoload.php’;` を記述すればcomposerの依存関係を自動的に読み込める

Helpers/RandomGenerator.php

```php
<?php

namespace Helpers;

use Faker\Factory;

class RandomGenerator {
    public static function user(): User {
        $faker = Factory::create();

        return new User(
            $faker->randomNumber(),
            $faker->firstName(),
            $faker->lastName(),
            $faker->email,
            $faker->password,
            $faker->phoneNumber,
            $faker->address,
            $faker->dateTimeThisCentury,
            $faker->dateTimeBetween('-10 years', '+20 years'),
            $faker->randomElement(['admin', 'user', 'editor'])
        );
    }

    public static function users(int $min, int $max): array {
        $faker = Factory::create();
        $users = [];
        $numOfUsers = $faker->numberBetween($min, $max);

        for ($i = 0; $i < $numOfUsers; $i++) {
            $users[] = self::user();
        }

        return $users;
    }
}
?>
```

ランダムなユーザーを生成して、動的にHTMLコンテンツを生成

```php
<?php
// コードベースのファイルのオートロード
spl_autoload_extensions(".php"); 
spl_autoload_register();

// composerの依存関係のオートロード
require_once 'vendor/autoload.php';

// クエリ文字列からパラメータを取得
$min = $_GET['min'] ?? 5;
$max = $_GET['max'] ?? 20;

// パラメータが整数であることを確認
$min = (int)$min;
$max = (int)$max;

// ユーザーの生成
$users = RandomGenerator::users($min, $max);
?>

<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>User Profiles</title>
    <style>
        /* ユーザーカードのスタイル */
    </style>
</head>
<body>
    <h1>User Profiles</h1>

    <?php foreach ($users as $user): ?>
    <div class="user-card">
        <!-- ユーザー情報の表示 -->
    </div>
    <?php endforeach; ?>

</body>
</html>
```

## Restaurant Chain Mockup

- compose.jsonを編集した後にcompose installで依存関係入れ忘れて表示させられずに時間溶けたので、更新のし忘れに気を付けたい

[https://github.com/SouthernMinami/restaurant-chain-mockup](https://github.com/SouthernMinami/restaurant-chain-mockup)

---

# 2/4

目標：モックアップの拡張（ダウンロード機能の追加）

## Download Lorem Ipsum

- 生成したいユーザー数と出力形式(HTML, Markdown, …)を指定できるフォーム追加を`generate.php`に追加
- ↑からPOSTリクエストを`download.php`に送信

```php
<form action="/submit-data" method="post">
<label for="name">Name:</label>
    <input type="text" id="name" name="name">
    <input type="submit" value="Submit">
</form>
```

`generate.php`

```php
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Generate Users</title>
</head>
<body>
    <form action="download.php" method="post">
        <label for="count">Number of Users:</label>
        <input type="number" id="count" name="count" min="1" max="100" value="5">
        
        <label for="format">Download Format:</label>
        <select id="format" name="format">
            <option value="html">HTML</option>
            <option value="markdown">Markdown</option>
            <option value="json">JSON</option>
            <option value="txt">Text</option>
        </select>

        <button type="submit">Generate</button>
    </form>
</body>
</html>
```

`donwload.php`

```php
<?php
require_once 'vendor/autoload.php';
require_once 'User.php';
require_once 'RandomGenerator.php';

// POSTリクエストからパラメータを取得
$count = $_POST['count'] ?? 5;
$format = $_POST['format'] ?? 'html';

// パラメータが正しい形式であることを確認
$count = (int)$count;

// ユーザーを生成
$users = RandomGenerator::users($count, $count);

if ($format === 'markdown') {
    header('Content-Type: text/markdown');
    header('Content-Disposition: attachment; filename="users.md"');
    foreach ($users as $user) {
        echo $user->toMarkdown();
    }
} elseif ($format === 'json') {
    header('Content-Type: application/json');
    header('Content-Disposition: attachment; filename="users.json"');
    $usersArray = array_map(fn($user) => $user->toArray(), $users);
    echo json_encode($usersArray);
} elseif ($format === 'txt') {
    header('Content-Type: text/plain');
    header('Content-Disposition: attachment; filename="users.txt"');
    foreach ($users as $user) {
        echo $user->toString();
    }
} else {
    // HTMLをデフォルトに
    header('Content-Type: text/html');
    foreach ($users as $user) {
        echo $user->toHTML();
    }
}
```

*別の方法：[file_get_contents](https://www.php.net/manual/en/function.file-get-contents.php)を利用した低レベルからのリクエスト

```php
<?php
    // Content-Typeをテキスト形式に設定
    header('Content-Type: text/plain');

    // ウェブサーバがPHPに渡すHTTPリクエストを取得
    $rawPostData = file_get_contents('php://input');

    // 生のPOSTデータをそのまま出力
    echo $rawPostData;
?>
```

https://github.com/SouthernMinami/restaurant-chain-mockup

本番環境へデプロイの前に

- node_modulesフォルダのファイルをignoreするように、膨大な依存関係やライブラリを含む`vendor`フォルダのファイルは`.gitignore`に入れてリポジトリにpushしないのが一般的

`vendor/`

[参考：.gitignoreの書きかた](https://qiita.com/inabe49/items/16ee3d9d1ce68daa9fff)

- `composer install` → push
- sshキーペア生成→githubに登録
- 本番環境でclone（*SSHで！） → リポジトリのルートフォルダで`composer install`

### デプロイ

- nginxインストール
- サブドメイン　userlorem.kano.wiki
- サイト公開用フォルダ　`mkdir -p /var/www/userlorem/public` → `sudo chown -R $USER:$USER /var/www/userlorem/public` (publicの所有者を現在のユーザーに移す)
- プロジェクトフォルダとpublicフォルダをリンク `sudo ln -s ~/dev/restaurant-chain-mockup  /var/www/userlorem/public`
- 設定ファイル　`sudo nano /etc/nginx/sites-available/userlorem.kano.wiki`

リバースプロキシの設定を含む

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/userlorem/public/restaurant-chain-mockup;
    index generate.php;

    server_name userlorem.kano.wiki;

    location / {
			try_files $uri $uri/ $uri.html $uri.php$is_args$query_string;    
		}

		location ~ \.php$ {
	    include snippets/fastcgi-php.conf;
	    fastcgi_pass unix:/run/php/php8.2-fpm.sock;
	}
}
```

Nginxに来たPHP関連のリクエストが、PHPコードを実行できるFastCGIサーバーに転送されるようになる

*適用するためにはhttps接続をサーバーポート443、それ以外の場合は80を使う必要がある

*PHPのバージョンを合わせることに注意

- DNS設定
- 設定ファイルへのショートカット　`sudo ln -s /etc/nginx/sites-available/userlorem.kano.wiki /etc/nginx/sites-enabled/`
- Nginxへのアクセスを許可 `chmod 755 ~`
- `sudo reboot`→再接続→`sudo systemctl start nginx`
- certbotでhttps接続設定

ERROR：Nginxのデフォルトページにつながってしまう＆certbotを起動できない（Nginx自体はエラーを出さずに起動できている）

対処

1. 次に取り掛かるMarkdown Converter用仮プロジェクトのindex.phpのデプロイを同じ手順で試してみる
2. 同じくうまくいかなかったら、EC2の設定などまだ見てないところを逐次確認していく
3. Markdown Converterのほうが成功したら、本プロジェクトとのデプロイ手順の差がどこにあるのか確認する

---

# 2/5

目標：モックアップのスタイル改善、デプロイ完了

- デフォルトページしか表示されなかった原因：EC2のセキュリティグループの設定で80, 443番ポートの使用を許可してなかった
- certbotで鍵作成→nginx再起動
- →今度はディレクトリ名タイポで404 → 修正してデプロイ成功

[https://userlorem.kano.wiki/](https://userlorem.kano.wiki/)

---

# 2/6

目標： Markdown Converter　最低限の機能実装

---

# 2/8 ~

*いつの間にかインスタンスAのアプリをデプロイ中にインスタンスBからデプロイしてたアプリが表示できなくなってた→片方のインスタンスに一旦全部移す

目標：UML図作成アプリ 全要件

monacoエディタとPlant UMLでUML図作成アプリ作る

機能要件

- ユースケース、クラス図、アクティビティ図、状態図、マインドマップ、ガントチャートをエディタからPlantUML構文で作れるようにする
- 変更後すぐに反映されるプレビュー
- .png, .svg, .txtのダウンロード機能
- Plant UML構文のチートシートページ
- 図作成の課題問題を用意する　JSON形式で作成してそれを読み取る
- 課題ページとは別のプレイグランドページも用意

非機能要件

- 将来的にいろんな種類の図表作成に対応できるように拡張性を持たせる　OOP
- ↑具体的には、問題の種類とダウンロードできるファイル形式の種類
- 作成済みの使わなくなったファイルは削除するようにしてストレージ節約する
- 本番環境デプロイ

[PHPからPlant UMLを使う](https://plantuml.com/ja-dark/code-php)

---
