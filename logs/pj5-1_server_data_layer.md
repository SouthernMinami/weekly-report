# 作業ログ：PJ5 サーバとデータ層

### 

My SQLセットアップ(DBeaver)

My SQLインストール　[https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04-ja](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04-ja)

CLIからMy SQL用ユーザー作成

- `mysql -u root -p`
- `CREATE USER 'vboxuser'@'localhost' IDENTIFIED BY 'UBU...';`
- `GRANT ALL PRIVILEGES ON *.* TO 'vboxuser'@'localhost';`

%だとシンタクスエラーになったので*を使用

DBeaverからMy SQLの接続テスト

- 左上のコンセントマークをクリック
- My SQLを選択
- 
- Server Host : localhost
- Port : 3306
- ユーザー名 : vboxuser
- パスワード：UBU…

* **Public Key Retrieval is not allowed**

↓以下が原因

[https://okuyan-techdiary.com/mysql-dbeaver-error/](https://okuyan-techdiary.com/mysql-dbeaver-error/)

新しいDBを追加

- `mysql -u vboxuser -p UBU…`
- `CREATE DATABASE practice_db;`
- ↑は `CREATE DATABASE practice_dp CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci`; と同等

---

envファイル

- .envは、認証情報、パスワード、APIキーなどの機密情報を含み、gitignoreに追加してアップロードしないようにするのが一般的
- 開発環境ではダミーの情報、本番環境では本番用の違う設定を持たせる

```
DATABASE_NAME="practice_db"
DATABASE_USER="vboxuser"
DATABASE_USER_PASSWORD="UBU..."
```

header.phpファイル内で.envファイルを読み取る

```php
<?php
const ENV_PATH = '.env';

class ReadAndParseEnvException extends Exception
{
}
;

/**
 * @throws ReadAndParseEnvException
 */
function env(string $pair): string
{
    $config = parse_ini_file(dirname(__FILE__) . '/' . ENV_PATH);

    if ($config === false) {
        throw new ReadAndParseEnvException();
    }

    return $config[$pair];
}

printf("Local database username: %s\n", env('DATABASE_USER'));
printf("Local database password (hashed): %s\n", password_hash(env('DATABASE_USER_PASSWORD'), PASSWORD_DEFAULT));

```

↓/header.phpにアクセスし、表示された

Local database username: vboxuser Local database password (hashed): 

…..(ハッシュ化されたパスワード）

---

ディレクトリ構造

後々クラスが増えてもコード管理とリファクタリングを容易にするため、ディレクトリ構成を決めておく

```
└── Exceptions
│   ├── ReadAndParseEnvException.php
├── Helpers
│   ├── Settings.php
└── init-app.php
```

Exeptions\ReadAndParseEnvException.php

```php
<?php

namespace Exceptions;

use Exception;

class ReadAndParseEnvException extends Exception{};

```

Helpers\Settings.php

```php
<?php

namespace Helpers;

use Exceptions\ReadAndParseEnvException;

class Settings{
    private const ENV_PATH =  '.env';

    public static function env(string $pair): string{
        // dirname関数は、指定されたファイルパスの親ディレクトリのパスを返す関数です。この関数には「levels」という整数型のパラメータを設定することができ、これは「いくつ親ディレクトリを遡るか」を指定するものです。デフォルトではこの「levels」は1に設定されており、つまり、ファイルの直接の親ディレクトリのパスを返します。
        $config = parse_ini_file( dirname(__FILE__, 2) . '/' . self::ENV_PATH);

        if($config === false){
            throw new ReadAndParseEnvException();
        }

        return $config[$pair];
    }
}

```

init-app.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    // クラス名から名前空間を取得
    $namespace = explode('\\', $class);
    // ファイルのパスを生成
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

use Helpers\Settings;

printf("Local database username: %s\n", Settings::env('DATABASE_USER'));
printf("Local database password (hashed): %s\n", password_hash(Settings::env('DATABASE_USER_PASSWORD'), PASSWORD_DEFAULT));

```

---

mysqliクラス

- LinuxベースのサーバにデフォルトでインストールされているPHPには、DB操作に使えるmysqliクラスが含まれる
- 初期化時に接続が行われ、mysqli_report()を呼び出し、接続失敗時にエラーを投げる
- mysql_connect()…接続成功時はmysqliインスタンスを、失敗時はfalseを返す

MySQLに接続

- sudo service mysql start
- init-app.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register();

use Helpers\Settings;
/*
    接続の失敗時にエラーを報告し、例外をスローします。データベース接続を初期化する前にこの設定を行ってください。
    テストするには、.env設定で誤った情報を入力します。
*/
mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);

/*
 * https://www.php.net/manual/en/class.mysqli.php で利用可能なすべてのメソッドを確認できます。
 */
$mysqli = new mysqli('localhost', Settings::env('DATABASE_USER'), Settings::env('DATABASE_USER_PASSWORD'), Settings::env('DATABASE_NAME'));

// https://www.php.net/manual/en/mysqli.get-charset.php
$charset = $mysqli->get_charset();

if($charset === null) throw new Exception('Charset could be read');

// データベースの文字セット、照合順序、統計情報について取得します。
printf(
    "%s's charset: %s.%s",
    Settings::env('DATABASE_NAME'),
    $charset->charset,
    PHP_EOL
);

printf(
    "collation: %s.%s",
    $charset->collation,
    PHP_EOL
);

// 接続を閉じるには、closeメソッドを使用します。
$mysqli->close();

```

- mysqliクラスを拡張してデフォルトコンストラクタを上書きし、接続パラメータをまとめて管理

Database/MySQLWrapper.php

```php
<?php

namespace Database;

use mysqli;
use Helpers\Settings;

class MySQLWrapper extends mysqli{
    public function __construct(?string $hostname = 'localhost', ?string $username = null, ?string $password = null, ?string $database = null, ?int $port = null, ?string $socket = null)
    {
        /*
            接続の失敗時にエラーを報告し、例外をスローします。データベース接続を初期化する前にこの設定を行ってください。
            テストするには、.env設定で誤った情報を入力します。
        */
        mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);

        $username = $username??Settings::env('DATABASE_USER');
        $password = $password??Settings::env('DATABASE_USER_PASSWORD');
        $database = $database??Settings::env('DATABASE_NAME');

        parent::__construct($hostname, $username, $password, $database, $port, $socket);
    }

    // クエリが問い合わせられるデフォルトのデータベースを取得します。
    // エラーは失敗時にスローされます（つまり、クエリがfalseを返す、または取得された行がない）
    // これらに対処するためにifブロックやcatch文を使用することもできます。
    public function getDatabaseName(): string{
        return $this->query("SELECT database() AS the_db")->fetch_row()[0];
    }
}

```

init-app.phpにアクセスすると、DBの情報が表示される

practice_db'utf8mb4 charsetcollation: utf8mb4_0900_ai_ci' 

---

DDL操作

- 通常、１つのアプリケーションに１つのデータベースが紐づく
- DDL操作…テーブルの定義（スキーマ）を管理、更新すること

Carsテーブル

| フィールド | タイプ | 説明 |
| --- | --- | --- |
| id | INT | 自動インクリメントID |
| make | VARCHAR(50) | 車のメーカー |
| model | VARCHAR(50) | 車のモデル |
| year | INT | 製造年 |
| color | VARCHAR(20) | 車の色 |
| price | FLOAT | 車の価格 |
| mileage | FLOAT | 走行距離 |
| transmission | VARCHAR(20) | トランスミッションの種類 |
| engine | VARCHAR(20) | エンジンの種類 |
| status | VARCHAR(10) | 状態（新品/中古/サルベージ/全損） |

- `USE practice_db;` 作業対象のDBを選択
- `practice_db`で使うテーブルを作成

```sql
CREATE TABLE carsTable (
  id INT PRIMARY KEY AUTO_INCREMENT,
  make VARCHAR(50),
  model VARCHAR(50),
  year INT,
  color VARCHAR(20),
  price FLOAT,
  mileage FLOAT,
  transmission VARCHAR(20),
  engine VARCHAR(20),
  status VARCHAR(10)
);

```

- SELECT * FROM carsTable;　まだデータが入ってないため、空のテーブルが返される
- dbeaverでＤＢの状態を保存すると、carsTableの追加が反映された

phpからクエリを実行してテーブル作成

- 一旦carsTableを削除 `DROP TABLE carsTable;`

Database\setup.php

```php
<?php
use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

$result = $mysqli->query("
    CREATE TABLE IF NOT EXISTS cars (
      id INT PRIMARY KEY AUTO_INCREMENT,
      make VARCHAR(50),
      model VARCHAR(50),
      year INT,
      color VARCHAR(20),
      price FLOAT,
      mileage FLOAT,
      transmission VARCHAR(20),
      engine VARCHAR(20),
      status VARCHAR(10)
    );
");

if($result === false) throw new Exception('Could not execute query.');
else print("Successfully ran all SQL setup queries.".PHP_EOL);

```

init-app.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    // クラス名から名前空間を取得
    $namespace = explode('\\', $class);
    // ファイルのパスを生成
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

use Database\MySQLWrapper;

// getoptはCLIで渡された指定された引数のオプションです。値のペアの配列を返します。
// 値が渡されない場合 (例：--myArg=123、値は123) は、値はfalseになります。issetを使用してそれが存在するかどうかをチェックします。
// short_optionsは、短いオプションの文字の配列を表す文字列を取り入れます。例えばabcは -a -b -c のオプションをチェックします。ロングオプションはオプションの完全な名前です。
$opts = getopt('',['migrate']);
if(isset($opts['migrate'])){
    printf('Database migration enabled.');
    // includeはPHPファイルをインクルードして実行します
    include('Database/setup.php');
    printf('Database migration ended.');
}

$mysqli = new MySQLWrapper();

$charset = $mysqli->get_charset();

if($charset === null) throw new Exception('Charset could be read');

// データベースの文字セット、照合順序、および統計に関する情報を取得します。
printf(
    "%s's charset: %s.%s",
    $mysqli->getDatabaseName(),
    $charset->charset,
    PHP_EOL
);

printf(
    "collation: %s.%s",
    $charset->collation,
    PHP_EOL
);

// 接続を閉じるには、closeメソッドが使用されます。
$mysqli->close();

```

- `php init-app.php --migrate`

Database migration started
SQL セットアップクエリを実行しました
Database migration finished
practice_db's charset: utf8mb4.
collation: utf8mb4_0900_ai_ci.

- dbeaverにもcarsTableの作成が反映された

---

Car Parts（課題）

- 基本は１対多　（Car: Parts)

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled.png)

- 多対多（Cars: Parts)の場合は、多様な異なる車が様々な種類の部品を持てるように、中間テーブル（CarPart)を設ける

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%201.png)

setup.php

```php
<?php

use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

// Carテーブル
$car_create_query = "
CREATE TABLE IF NOT EXISTS Car (
id INT PRIMARY KEY AUTO_INCREMENT,
make VARCHAR(50),
model VARCHAR(50),
year INT,
color VARCHAR(20),
price FLOAT,
milegage FLOAT,
transmission VARCHAR(20),
engine VARCHAR(20),
status VARCHAR(10)
);
";

$result = $mysqli->query($car_create_query);

if (!$result)
    throw new Exception('Carテーブルのセットアップクエリを実行できませんでした');
else
    print("Carテーブルのセットアップクエリを実行しました" . PHP_EOL);

$part_create_query = "
    CREATE TABLE IF NOT EXISTS Part (
        id INT PRIMARY KEY AUTO_INCREMENT,
        name VARCHAR(50),
        description TEXT,
        price FLOAT,
        quantityInStock INT
    );
";

$result = $mysqli->query($part_create_query);

if (!$result)
    throw new Exception('Partテーブルのセットアップクエリを実行できませんでした');
else
    print("Partテーブルのセットアップクエリを実行しました" . PHP_EOL);

// CarPartテーブル
$car_part_create_query = "
    CREATE TABLE IF NOT EXISTS CarPart (
        carId INT, FOREIGN KEY(carId) REFERENCES Car(id),
        partId INT, FOREIGN KEY(partId) REFERENCES Part(id),
        quantity INT
    );
";

$result = $mysqli->query($car_part_create_query);
if (!$result)
    throw new Exception('CarPartテーブルのセットアップクエリを実行できませんでした');
else
    print("CarPartテーブルのセットアップクエリを実行しました" . PHP_EOL);

// Carのデータ
$car_insert_query = "
    INSERT INTO Car (make, model, year, color, price, milegage, transmission, engine, status) 
    VALUES 
        ('Toyota', 'Camry', 2018, 'White', 20000, 10000, 'Automatic', 'V6', 'New'), 
        ('Honda', 'Civic', 2017, 'Black', 15000, 8000, 'Automatic', 'V4', 'Used'), 
        ('Nissan', 'Altima', 2016, 'Red', 18000, 9000, 'Automatic', 'V6', 'Used'), 
        ('Ford', 'Fusion', 2015, 'Blue', 16000, 8500, 'Automatic', 'V4', 'Used')
";

$result = $mysqli->query($car_insert_query);
if (!$result)
    throw new Exception('Carテーブルのデータを挿入できませんでした');
else
    print("Carテーブルにデータを挿入しました" . PHP_EOL);

// Partのデータ
$part_insert_query = "
    INSERT INTO Part (name, description, price, quantityInStock)
    VALUES 
        ('Brake Pads', 'Front and Rear Brake Pads', 100, 50), 
        ('Oil Filter', 'Oil Filter for Engine', 10, 100), 
        ('Air Filter', 'Air Filter for Engine', 15, 100), 
        ('Headlight', 'Front Headlight', 50, 100)
";

$result = $mysqli->query($part_insert_query);
if (!$result)
    throw new Exception('Partテーブルのデータを挿入できませんでした');
else
    print("Partテーブルにデータを挿入しました" . PHP_EOL);

// CarPartのデータ
$car_part_insert_query = "
    INSERT INTO CarPart (carId, partId, quantity)
    VALUES 
        (1, 1, 2), (1, 2, 3), (1, 3, 1), (1, 4, 2), 
        (2, 1, 1), (2, 2, 2), (2, 3, 1), (2, 4, 1), 
        (3, 1, 2), (3, 2, 2), (3, 3, 2), (3, 4, 2), 
        (4, 1, 1), (4, 2, 1), (4, 3, 1), (4, 4, 1)
";

$result = $mysqli->query($car_part_insert_query);
if (!$result)
    throw new Exception('CarPartテーブルのデータを挿入できませんでした');
else
    print("CarPartテーブルにデータを挿入しました" . PHP_EOL);

echo "データの挿入が完了しました" . PHP_EOL;

$mysqli->close();

```

↓リファクタリング

```php
<?php

use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

// Carテーブル
const CREATE_CAR_TABLE_QUERY = "
CREATE TABLE IF NOT EXISTS Car (
id INT PRIMARY KEY AUTO_INCREMENT,
make VARCHAR(50),
model VARCHAR(50),
year INT,
color VARCHAR(20),
price FLOAT,
milegage FLOAT,
transmission VARCHAR(20),
engine VARCHAR(20),
status VARCHAR(10)
);
";

// Partテーブル
const CREATE_PART_TABLE_QUERY = "
    CREATE TABLE IF NOT EXISTS Part (
        id INT PRIMARY KEY AUTO_INCREMENT,
        name VARCHAR(50),
        description TEXT,
        price FLOAT,
        quantityInStock INT
    );
";

// CarPartテーブル(中間テーブル)
const CREATE_CAR_PART_TABLE_QUERY = "
    CREATE TABLE IF NOT EXISTS CarPart (
        carId INT, FOREIGN KEY(carId) REFERENCES Car(id),
        partId INT, FOREIGN KEY(partId) REFERENCES Part(id),
        quantity INT
    );
";
function insertCarQuery(
    string $make,
    string $model,
    int $year,
    string $color,
    float $price,
    float $mileage,
    string $transmission,
    string $engine,
    string $status
): string {
    return sprintf("
        INSERT INTO Car (make, model, year, color, price, milegage, transmission, engine, status)
        VALUES ('%s', '%s', %d, '%s', %f, %f, '%s', '%s', '%s');
    ", $make, $model, $year, $color, $price, $mileage, $transmission, $engine, $status);
}
;

function insertPartQuery(
    string $name,
    string $description,
    float $price,
    int $quantityInStock
): string {
    return sprintf("
        INSERT INTO Part (name, description, price, quantityInStock)
        VALUES ('%s', '%s', %f, %d);
    ", $name, $description, $price, $quantityInStock);
}

function insertCarPartQuery(
    int $carId,
    int $partId,
    int $quantity
): string {
    return sprintf("
        INSERT INTO CarPart (carId, partId, quantity)
        VALUES (%d, %d, %d);
    ", $carId, $partId, $quantity);
}

function runQuery(mysqli $mysqli, string $query): void
{
    $result = $mysqli->query($query);
    if (!$result) {
        throw new Exception('クエリを実行できませんでした');
    } else {
        print("クエリを実行しました" . PHP_EOL);
    }
}

// Carのデータ
runQuery($mysqli, CREATE_CAR_TABLE_QUERY);
runQuery(
    $mysqli,
    insertCarQuery(
        'Toyota',
        'Camry',
        2018,
        'White',
        20000,
        10000,
        'Automatic',
        'V6',
        'New'
    )
);

// Partのデータ
$part_insert_query = "
    INSERT INTO Part (name, description, price, quantityInStock)
    VALUES 
        ('Brake Pads', 'Front and Rear Brake Pads', 100, 50), 
        ('Oil Filter', 'Oil Filter for Engine', 10, 100), 
        ('Air Filter', 'Air Filter for Engine', 15, 100), 
        ('Headlight', 'Front Headlight', 50, 100)
";
runQuery($mysqli, CREATE_PART_TABLE_QUERY);
runQuery(
    $mysqli,
    insertPartQuery(
        'Brake Pads',
        'Front and Rear Brake Pads',
        100,
        50
    )
);

// CarPartのデータ
$car_part_insert_query = "
    INSERT INTO CarPart (carId, partId, quantity)
    VALUES 
        (1, 1, 2), (1, 2, 3), (1, 3, 1), (1, 4, 2), 
        (2, 1, 1), (2, 2, 2), (2, 3, 1), (2, 4, 1), 
        (3, 1, 2), (3, 2, 2), (3, 3, 2), (3, 4, 2), 
        (4, 1, 1), (4, 2, 1), (4, 3, 1), (4, 4, 1)
";
runQuery($mysqli, CREATE_CAR_PART_TABLE_QUERY);
runQuery(
    $mysqli,
    insertCarPartQuery(
        1,
        1,
        2
    )
);

echo "データの挿入が完了しました" . PHP_EOL;

$mysqli->close();

```

---

ER図(Entity Relationship Diagram)

- エンティティ間の多重度と依存関係を表す
- 〇の場合はオプション、〇がなければ必須の依存関係

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%202.png)

---

スキーマスクリプティング

.sqlファイルに直接テーブルのスキーマ設定を書きこむ（手動スクリプト）

Database/Examples/cars-setup1.sql

```sql
CREATE TABLE IF NOT EXISTS cars1 (
    id INT PRIMARY KEY AUTO_INCREMENT,
    make VARCHAR(50),
    model VARCHAR(50),
    year INT,
    color VARCHAR(20),
    price FLOAT,
    mileage FLOAT,
    transmission VARCHAR(20),
    engine VARCHAR(20),
    status VARCHAR(10)
);

```

Database/setup1.php

```php

<?php
use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

$result = $mysqli->query(file_get_contents(__DIR__ . '/Examples/cars-setup1.sql'));

if($result === false) throw new Exception('Could not execute query.');
else print("Successfully ran all SQL setup queries.".PHP_EOL);

```

init-app.php

```php
<?php
use Database\MySQLWrapper;

spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    // クラス名から名前空間を取得
    $namespace = explode('\\', $class);
    // ファイルのパスを生成
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

// CLIから渡したmigrateオプションを取得
$ops = getopt('', ['migrate']);
if (isset($ops['migrate'])) {
    printf('Database migration started' . PHP_EOL);
    include('Database/setup.php');
    include('Database/setup1.php'); // 追加
    printf('Database migration finished' . PHP_EOL);
}

$mysqli = new MySQLWrapper();
$charset = $mysqli->get_charset();
if ($charset === null)
    throw new Exception('Charsetがデータベースから読み取れませんでした。');

printf(
    "%s's charset: %s.%s",
    $mysqli->getDatabaseName(),
    $charset->charset,
    PHP_EOL
);

printf(
    "collation: %s.%s",
    $charset->collation,
    PHP_EOL
);

$mysqli->close();

```

MySQLのCLIで、直接SQLを打ち込む代わりに.sqlファイルのパスを書き込み、マイグレーション

```bash
mysql -u vboxuser -p practice_db < Database/Examples/cars-setup1.sql

php init-app.php --migrate
```

メリット

- 直観的でわかりやすい
- 小規模プロジェクトに適している
- 大規模プロジェクトでも、複数人での管理の容易さから適してる場合がある

課題①：ブログ投稿サービスのDB

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%203.png)

setup.php

```php
<?php

use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

function insertUserQuery(string $username, string $email, string $password, string $email_confirmed_at, string $created_at, string $updated_at): string
{
    return sprintf("
        INSERT INTO user (username, email, password, email_confirmed_at, created_at, updated_at)
        VALUES ('%s', '%s', '%s', '%s', '%s', '%s');
    ", $username, $email, $password, $email_confirmed_at, $created_at, $updated_at);
}

function insertPostQuery(string $title, string $content, string $created_at, string $updated_at): string
{
    return sprintf("
        INSERT INTO post (title, content, created_at, updated_at)
        VALUES ('%s', '%s', '%s', '%s');
    ", $title, $content, $created_at, $updated_at);
}

function insertCommentQuery(string $comment_text, string $created_at, string $updated_at): string
{
    return sprintf("
        INSERT INTO comment (comment_text, created_at, updated_at)
        VALUES ('%s', '%s', '%s');
    ", $comment_text, $created_at, $updated_at);
}

function insertPostLikeQuery(int $user_id, int $post_id): string
{
    return sprintf("
        INSERT INTO post_like (user_id, post_id)
        VALUES (%d, %d);
    ", $user_id, $post_id);
}
function
    insertCommentLikeQuery(
    int $user_id,
    int $comment_id
): string {
    return sprintf("
        INSERT INTO comment_like (user_id, comment_id)
        VALUES (%d, %d);
    ", $user_id, $comment_id);
}

function runQuery(mysqli $mysqli, string $query): void
{
    $result = $mysqli->query($query);
    if (!$result) {
        throw new Exception('クエリを実行できませんでした。');
    } else {
        print('クエリを実行しました。');
    }
}

runQuery(
    $mysqli,
    insertUserQuery(
        '新しいユーザー',
        '新しいユーザーのメールアドレス',
        '新しいユーザーのパスワード',
        '2024-01-01 00:00:00',
        '2024-01-01 00:00:00',
        '2024-01-01 00:00:00'
    )
);

runQuery(
    $mysqli,
    insertPostQuery(
        '新しい投稿',
        '新しい投稿の内容',
        '2024-01-01 00:00:00',
        '2024-01-01 00:00:00'
    )
);

runQuery(
    $mysqli,
    insertCommentQuery(
        '新しいコメント',
        '2024-01-01 00:00:00',
        '2024-01-01 00:00:00'
    )
);

runQuery(
    $mysqli,
    insertPostLikeQuery(1, 1)
);

runQuery(
    $mysqli,
    insertCommentLikeQuery(1, 1)
);

```

user.sql

```sql
CREATE TABLE IF NOT EXISTS user (
    id INT AUTO_INCREMENT PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(50) NOT NULL,
    password VARCHAR(255) NOT NULL,
    email_confirmed_at TIMESTAMP,
    created_at TIMESTAMP,
    updated_at TIMESTAMP
) 
```

post.sql

```sql
CREATE TABLE IF NOT EXISTS  post (
    post_id INT AUTO_INCREMENT PRIMARY KEY,
    title VARCHAR(50) NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    user_id INT, FOREIGN KEY (user_id) REFERENCES user(id)
)
```

comment.sql

```sql
CREATE TABLE IF NOT EXISTS  comment (
    comment_id INT AUTO_INCREMENT PRIMARY KEY,
    comment_text TEXT NOT NULL,
    created_at TIMESTAMP,
    updated_at TIMESTAMP,
    user_id INT, FOREIGN KEY (user_id) REFERENCES user(id),
    post_id INT, FOREIGN KEY (post_id) REFERENCES post(post_id)
)
```

post_like.sql

```sql
CREATE TABLE IF NOT EXISTS  post_like ( 
    user_id INT, FOREIGN KEY (user_id) REFERENCES user(id),
    post_id INT, FOREIGN KEY (post_id) REFERENCES post(post_id),
    PRIMARY KEY (user_id, post_id)
)
```

comment_like.sql

```sql
CREATE TABLE IF NOT EXISTS  comment_like (
    user_id INT, FOREIGN KEY (user_id) REFERENCES user(id),
    comment_id INT, FOREIGN KEY (comment_id) REFERENCES comment(comment_id),
    PRIMARY KEY (user_id, comment_id)
)

```

CLIの操作（マイグレーション）は↑のスキーマスクリプティングを参照

課題②：投稿に関する機能拡張

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%204.png)

- 新しいテーブル…先ほどと同様にCREATE TABLE IF NOT EXISTS … で作成
- 既存テーブルの新カラム…`ALTER TABLE … ADD …` で更新

---

スキーマ管理（マイグレーションベース）・コマンドの拡張

マイグレーションベースのスキーマ管理…主にコマンドを用いて、小規模な変更を段階的にスキーマに適用していく方法

特徴

- スキーマの変更履歴は追跡され、各マイグレーションスクリプトはバージョン管理される。これにより、任意の状態に進んだり戻ったりできる
- スキーマの変更はスタック方式なため、最後に加えられた変更から順に１つずつロールバックが可能
- すべてのサーバスキーマは一貫したマイグレーションスクリプトの順序に従って更新されるので、あるサーバが数回のマイグレーション分遅れてたとしても一方のサーバと同じ状態に同期できる

コマンド作成のサンプルスクリプト

- `CREATE DATABASE table_migrations;`
- スクリプト格納用のファイルの形式

{YYYY-MM-DD}{UNIX_TIMESTAMP}{FILENAME}.php

- クラス/インタフェース

Commands/Command.php

```php
<?php

// すべてのコマンドが持っているメソッドを定義するインタフェース

namespace Commands;

interface Command
{
    // コマンドのエイリアスを取得
    public static function getAlias(): string;
    // コマンドの引数の配列を取得
    /** @return Argument[] */
    public static function getArgs(): array;
    // コマンドに関するヘルプを取得
    public static function getHelp(): string;
    // 値が必要かどうかを取得
    public static function isCommandValueRequired(): bool;

    // 引数の値を取得
    /** @return bool | string */
    public function getArgValue(string $arg): bool|string;
    // コマンドを実行
    public function execute(): int;
}
```

Commands/AbstractCommand.php

```php
<?php

// このクラスを拡張してコマンドを作成する
// 子クラスに stdout に出力する log() メソッドや引数オプションを取得する方法などのヘルパーメソッドを含む
// シェルから渡された引数を解析し、引数のマップを作成するのが目的

namespace Commands;

use Exception;

abstract class AbstractCommand implements Command
{
    protected ?string $value;
    protected array $argsMap = [];
    protected static ?string $alias = null;

    protected static bool $commandValueRequired = false;

    /**
     * @throws Exception
     */

    public function __construct()
    {
        $this->setUpArgsMap();
    }

    // シェルから引数を読み込み、引数のハッシュマップをセットアップする
    private function setUpArgsMap(): void
    {
        // グローバルスコープの引数の配列を取得
        $args = $GLOBALS['argv'];
        // エイリアスのインデックスを検索
        $startIndex = array_search($this->getAlias(), $args);

        // エイリアスが見つからない場合は例外をスロー
        // それ以外の場合は、エイリアスのインデックスをインクリメント
        if ($startIndex === false)
            throw new Exception(sprintf("%sというエイリアスが見つかりませんでした。", $this->getAlias()));
        else
            $startIndex++;

        $shellArgs = [];

        // エイリアスの次の引数が存在しないか、次の引数がオプション(-)の場合で、
        // コマンドの値が必要な場合は例外をスロー
        // それ以外の場合は、エイリアスの次の引数を引数マップに追加し、インデックスをインクリメント
        if (!isset ($args[$startIndex]) || $args[$startIndex][0] === '-') {
            if ($this->isCommandValueRequired()) {
                throw new Exception(sprintf("%sコマンドを実行するには値を入力してください。", $this->getAlias()));
            }
        } else {
            $this->argsMap[$this->getAlias()] = $args[$startIndex];
            $startIndex++;
        }

        // すべての引数を$argsに格納
        for ($i = $startIndex; $i < count($args); $i++) {
            $arg = $args[$i];

            // ハイフンがある場合、ハイフンをキーとして扱う
            if ($arg[0] . $arg[1] === '--')
                $key = substr($arg, 2);
            else if ($arg[0] === '-')
                $key = substr($arg, 1);
            else
                throw new Exception('オプションは-か--で始まる必要があります。');

            $shellArgs[$key] = true;

            // 次のargsエントリがオプション(-)でない場合は引数値として扱い、shellArgsマップに保存
            if (isset ($args[$i + 1]) && $args[$i + 1] !== '-') {
                $shellArgs[$key] = $args[$i + 1];
                $i++;
            }
        }

        // コマンドの引数マップを設定
        foreach ($this->getArgs() as $arg) {
            $argString = $arg->getArg();
            $value = null;

            if ($arg->isShortAllowed() && isset ($shellArgs[$argString[0]]))
                $value = $shellArgs[$argString[0]];
            else if (isset ($shellArgs[$argString]))
                $value = $shellArgs[$argString];

            if ($value === null) {
                if ($arg->isRequired())
                    throw new Exception(sprintf("必要な引数%sが見つかりませんでした。", $argString));
                else
                    $this->argsMap[$argString] = false;
            } else
                $this->argsMap[$argString] = $value;
        }

        // マップをログに出力
        $this->log(json_encode($this->argsMap));
    }

    public static function getHelp(): string
    {
        $helpString = "Command: " . static::getAlias() . (static::isCommandValueRequired() ? " {value}" : "") . PHP_EOL;

        $args = static::getArgs();
        if (empty ($args))
            return $helpString;

        $helpString .= "Arguments: " . PHP_EOL;

        foreach ($args as $arg) {
            $helpString .= " --" . $arg->getArg();

            if ($arg->isShortAllowed()) {
                $helpString .= " (-" . $arg->getArg()[0] . ")";
            }
            $helpString .= ": " . $arg->isRequired() ? " (Required)" : " (Optional)";
            $helpString .= PHP_EOL;
        }

        return $helpString;
    }

    public static function getAlias(): string
    {
        // エイリアスが設定されてない場合はクラス名を返す
        return static::$alias != null ? static::$alias : static::class;
    }

    public static function isCommandValueRequired(): bool
    {
        return static::$commandValueRequired;
    }

    public function getCommandValue(): string
    {
        return $this->argsMap[static::getAlias()] ?? "";
    }

    public function getArgValue(string $arg): bool|string
    {
        return $this->argsMap[$arg];
    }

    protected function log(string $info): void
    {
        fwrite(STDOUT, $info . PHP_EOL);
    }

    /** @return Argument[] */
    public abstract static function getArgs(): array;
    public abstract function execute(): int;
}
```

Commands/Argument.php

```php
<?php

// コマンドが使える引数を定義する
// 必要に応じてオプション引数も追加

namespace Commands;

class Argument
{
    private string $arg;
    private string $description = '';
    private bool $required = true;
    private bool $allowAsShort = false;

    public function __construct(string $arg)
    {
        $this->arg = $arg;
    }

    public function getArg(): string
    {
        return $this->arg;
    }

    public function getDescription(): string
    {
        return $this->description;
    }

    // descriptionを設定し、引数自身を返す
    public function description(string $description): Argument
    {
        $this->description = $description;
        return $this;
    }

    // 引数が必須かどうかを返す
    public function isRequired(): bool
    {
        return $this->required;
    }

    // 引数が必須かどうかを設定し、引数自身を返す
    public function required(bool $required): Argument
    {
        $this->required = $required;
        return $this;
    }

    // 短縮形式の引数を許可するかどうかを返す
    public function isShortAllowed(): bool
    {
        return $this->allowAsShort;
    }

    // 短縮形式の引数を許可するかどうかを設定し、引数自身を返す
    public function allowAsShort(bool $allowAsShort): Argument
    {
        $this->allowAsShort = $allowAsShort;
        return $this;
    }
}
```

- コマンドのリスト用レジストリ

Commands/registry.php

```php
<?php

// コマンドを登録するためのレジストリ
// consoleはここから読み取る
return [
    Commands\Programs\Migrate::class,
    Commands\Programs\CodeGeneration::class,
];
```

- migration コマンド

Commands/Programs/Migrate.php

```php
<?php

// マイグレーションの実行、ロールバック、新しいスキーマインストールを行う
namespace Commands\Programs;

use Commands\AbstractCommand;
use Commands\Argument;

class Migrate extends AbstractCommand
{
    // 使用するコマンド名
    protected static ?string $alias = 'migrate';

    // 引数の割当
    public static function getArgs(): array
    {
        return [
            (new Argument('rollback'))->description('マイグレーションをロールバックします。ロールバック回数を指定することもできます。')->required(false)->allowAsShort(true),
        ];
    }

    public function execute(): int
    {
        $rollback = $this->getArgValue('rollback');
        if (!$rollback) {
            $this->log("マイグレーションを開始します。");
            $this->migrate();
        } else {
            $rollback = $rollback === true ? 1 : (int) $rollback;
            $this->log("マイグレーションをロールバックしています。");
            for ($i = 0; $i < $rollback; $i++) {
                $this->rollback();
            }
        }

        return 0;
    }

    private function migrate(): void
    {
        $this->log("Migrating...");
        $this->log("マイグレーションが完了しました。\n");
    }

    private function rollback(): void
    {
        $this->log("Rolling back...");
        $this->log("ロールバックが完了しました。\n");
    }
}
```

- code-genコマンド

```php
<?php

// コード生成のコマンド

namespace Commands\Programs;

use Commands\AbstractCommand;

class CodeGeneration extends AbstractCommand
{
    // 使用するコマンド名
    protected static ?string $alias = 'code-gen';
    protected static bool $requiredCommandValue = true;

    // 引数の割当
    public static function getArgs(): array
    {
        return [];
    }

    public function execute(): int
    {
        $codeGenType = $this->getCommandValue();
        $this->log('Generating code for.......' . $codeGenType);
        return 0;
    }
}

```

- コマンドプログラムのためのエントリポイント

console

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

$commands = include "Commands/registry.php";
// 第２引数（実際に実行するコマンド）
$inputCommand = $argv[1];

foreach ($commands as $commandClass) {
    $alias = $commandClass::getAlias();

    if ($inputCommand === $alias) {
        if (in_array('--help', $argv)) {
            fwrite(STDOUT, $commandClass::getHelp());
            exit (0);
        } else {
            $command = new $commandClass();
            $result = $command->execute();
            exit ($result);
        }
    }
}

fwrite(STDOUT, "コマンドを実行できませんでした。" . PHP_EOL);

```

- migrateの実行

`php console migrate`

```bash
{"rollback":false}
マイグレーションを開始します。
Migrating...
マイグレーションが完了しました。
```

`php console migrate --rollback` / `php console migrate -r`

```bash
{"rollback":true}
マイグレーションをロールバックしています。
Rolling back...
ロールバックが完了しました。
```

`php console migrate -r 3`

```bash
{"rollback":"3"}
マイグレーションをロールバックしています。
Rolling back...
ロールバックが完了しました。

Rolling back...
ロールバックが完了しました。

Rolling back...
ロールバックが完了しました。
```

- code-genコマンドの実行

`php console code-gen javascript`

```bash
{"code-gen":"javascript"}
Generating code for.......javascript
```

---

コマンドの拡張

課題①：db-wipeコマンド（DBのクリア）

- デフォルト… DBの内容をすべてクリアする
- dumpオプション… DBに対してmysqldumpを実行し、ダンプファイルにバックアップを作成した後で、DBをクリアする
- restoreオプション… ダンプファイルからDBの内容を復元する

DBWipe.php

```php
<?php

// DB全体をクリアするコマンド

namespace Commands\Programs;

use Commands\AbstractCommand;
use Commands\Argument;

class DBWipe extends AbstractCommand
{
    protected static ?string $alias = 'dbwipe';

    public static function getArgs(): array
    {
        return [
            (new Argument('dump'))->description('データベースをダンプし、ダンプファイルを作成します。')->required(false)->allowAsShort(true),
            (new Argument('restore'))->description('ダンプファイルからデータを復元します。')->required(false)->allowAsShort(true),
        ];
    }

    public function execute(): int
    {
        // ユーザー名を入力してください
        $username = readline('ユーザー名を入力してください: ');
        $dbname = readline('内容をクリアするデータベース名を入力してください: ');

        $dump = $this->getArgValue('dump');
        $restore = $this->getArgValue('restore');

        if ($dump) {
            $this->dump($username, $dbname);
        } else if ($restore) {
            $this->restore($username, $dbname);
            return 0;
        }
        $this->dbwipe($username, $dbname);
        return 0;
    }

    private function dbwipe(string $username, string $dbname): void
    {
        exec('mysql -u ' . $username . ' -p ' . $dbname . ' -e "DROP DATABASE IF EXISTS ' . $dbname . '; CREATE DATABASE ' . $dbname . ';"');
        $this->log(sprintf("データベース %s の内容がクリアされ、再作成されました。" . PHP_EOL, $dbname));
    }

    private function dump(string $username, string $dbname): void
    {
        exec('mysqldump -u ' . $username . ' -p ' . $dbname . ' > Database\Schema\backup.sql');
        $this->log(sprintf("データベース %s の内容からダンプファイルを作成しました。" . PHP_EOL, $dbname));
    }

    private function restore(string $username, string $dbname): void
    {
        exec('mysql -u ' . $username . ' -p ' . $dbname . ' < Database\Schema\backup.sql');
        $this->log(sprintf("データベース %s の内容をダンプファイルから復元しました。" . PHP_EOL, $dbname));
    }
}
```

Database/Schema/backup.sql (空)

- `php console dbwipe` → ユーザ名、DB名、パスワードの入力→ DBがDROP, CREATEされて空になった
- `php console dpwipe --dump` → 入力→ DBが空になり、backup.sqlを通してDBのバックアップが作成されている
- `php console dbwipe --restore` → 入力 → dumpした内容の通りにDBが復元された

課題②：book-search

課題③：command-generation (code-genを拡張）

---

マイグレーションベースのスキーマ管理

Database/SchemaMigration.php

```php
<?php

namespace Database;

interface SchemaMigration
{
    public function up(): array;
    public function down(): array;
}

```

Commands/Programs/CodeGeneration.php

```php
<?php

// コード生成のコマンド

namespace Commands\Programs;

use Commands\AbstractCommand;
use Commands\Argument;

class CodeGeneration extends AbstractCommand
{
    // 使用するコマンド名
    protected static ?string $alias = 'code-gen';
    protected static bool $requiredCommandValue = true;

    // 引数の割当
    public static function getArgs(): array
    {
        return [(new Argument('name'))->description('生成されるファイル名。')->required(false)];
    }

    public function execute(): int
    {
        $codeGenType = $this->getCommandValue();

        switch ($codeGenType) {
            case 'command':
                $this->generateCommand(readline("コマンド名を入力してください: "));
                break;
            case 'migration':
                $migrationName = $this->getCommandValue();
                $this->log(sprintf("マイグレーションファイル %s を生成します。", $migrationName));
                $this->generateMigrationFile($migrationName);
            default:
                $this->log('Invalid code generation type.');
                break;
        }
        return 0;
    }

    // マイグレーションファイルを生成する関数
    private function generateMigrationFile(string $migrationName): void
    {
        // {YYYY-MM-DD}{UNIXTIME}{ClassName}.phpのフォーマットでマイグレーションファイルを生成
        $filename = sprintf(
            '%s_%s_%s.php',
            date('Y-m-d'),
            time(),
            $migrationName
        );

        $migrationContent = $this->getMigrationContent($migrationName);

        // 移行先
        $path = sprintf("%s/../../Database/Migrations/%s", __DIR__, $filename);

        file_put_contents($path, $migrationContent);
        $this->log(sprintf("マイグレーションファイル %s が作成されました。", $filename));
    }

    // マイグレーションファイルの内容を取得する関数
    private function getMigrationContent(string $migrationName): string
    {
        $className = $this->pascalCase($migrationName);

        return <<<MIGRATION
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class {$className} implements SchemaMigration
{
    public function up(): array
    {
        // マイグレーション処理を書く
        return [];
    }

    public function down(): array
    {
        // ロールバック処理を書く
        return [];
    }
}
MIGRATION;

    }

    // スネークケースをパスカルケースに変換する関数
    private function pascalCase(string $snakeCase): string
    {
        return str_replace(' ', '', ucwords(str_replace('_', ' ', $snakeCase)));
    }

    // 新しいコマンドファイルを生成してProgramsに追加する関数
    private function generateCommand(string $name): void
    {
        $capitalized_name = ucfirst($name);
        // 空白区切りで引数を取得
        $args = explode(' ', readline("コマンドで利用するオプションをスペース区切りで入力してください:"));
        $exec_code = readline("コマンドの実行コードを入力してください: ");

        $file_path = "Commands/Programs/" . $capitalized_name . ".php";
        $content = "<?php
            namespace Commands\Programs;
            
            use Commands\AbstractCommand;
            use Commands\Argument;

            class $capitalized_name extends AbstractCommand
            {
                protected static ?string \$alias = '$name';

                public static function getArgs(): array
                {

                    return [
                        " . implode(",\n", array_map(function ($arg) {
            return "(new Argument('$arg'))->description('')->required(false)->allowAsShort(true)";
        }, $args)) . "
                    ];
                }

                public function execute(): int
                {
                    $exec_code
                    return 0;
                }
            }
        ";

        file_put_contents($file_path, $content);
        // registry.phpに新しいコマンドを追加
        $registry_path = "Commands/registry.php";
        $registry_content = file_get_contents($registry_path);
        $registry_content = str_replace("return [", "return [\n    Commands\Programs\\$capitalized_name::class,", $registry_content);
        file_put_contents($registry_path, $registry_content);
    }
}

```

usersテーブル

- `php console code-gen migration --name CreateUserTable1`

Database/Migrations/2024-…_CreateUserTable1.php　が生成された

編集

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreateUserTable1 implements SchemaMigration
{
    // マイグレーションファイルを生成後、メソッドの処理を記述する
    public function up(): array
    {
        // マイグレーション
        return [
            "CREATE TABLE users (
                id BIGINT PRIMARY KEY AUTO_INCREMENT,
                username VARCHAR(255) NOT NULL,
                email VARCHAR(255) NOT NULL UNIQUE,
                password VARCHAR(255) NOT NULL,
                email_confirmed_at VARCHAR(255),
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            )"
        ];
    }

    public function down(): array
    {
        // ロールバック処理を書く
        return [
            "DROP TABLE users"
        ];
    }
}
```

postsテーブル

Database/Migrations/2024-…_CreateUserTable1.php

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreatePostTable1 implements SchemaMigration
{
    public function up(): array
    {
        return [
            "CREATE TABLE posts (
                id INT PRIMARY KEY AUTO_INCREMENT,
                title VARCHAR(255) NOT NULL,
                content TEXT NOT NULL,
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP,
                user_id BIGINT,
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE
            )"
        ];
    }

    public function down(): array
    {
        return [
            "DROP TABLE posts"
        ];
    }
}

```

- migrateコマンドを編集

Commands/Programs/Migrate.php

```php
<?php

// マイグレーションの実行、ロールバック、新しいスキーマインストールを行う
namespace Commands\Programs;

use Commands\AbstractCommand;
use Commands\Argument;
use Database\MySQLWrapper;

class Migrate extends AbstractCommand
{
    // 使用するコマンド名
    protected static ?string $alias = 'migrate';

    // 引数の割当
    public static function getArgs(): array
    {
        return [
            (new Argument('rollback'))->description('マイグレーションをロールバックします。ロールバック回数を指定することもできます。')->required(false)->allowAsShort(true),
            (new Argument('init'))->description('新しいマイグレーションテーブルを作成します。')->required(false)->allowAsShort(true),
        ];
    }

    public function execute(): int
    {
        $rollback = $this->getArgValue('rollback');

        if ($this->getArgValue('init')) {
            $this->createMigrationsTable();
        }

        if (!$rollback) {
            $this->log("マイグレーションを開始します。");
            $this->migrate();
        } else {
            $rollback = $rollback === true ? 1 : (int) $rollback;
            $this->log("マイグレーションをロールバックしています。");
            for ($i = 0; $i < $rollback; $i++) {
                $this->rollback();
            }
        }

        return 0;
    }

    private function createMigrationsTable(): void
    {
        $this->log("マイグレーションテーブルを作成します。");

        $mysqli = new MySQLWrapper();

        $result = $mysqli->query("
            CREATE TABLE IF NOT EXISTS migrations (
                id INT AUTO_INCREMENT PRIMARY KEY,
                filename VARCHAR(255) NOT NULL
            );
        ");

        if (!$result) {
            throw new \Exception("マイグレーションテーブルの作成に失敗しました。");
        }

        $this->log("マイグレーションテーブルの作成が完了しました。");
    }

    private function migrate(): void
    {
        $this->log("Migrating...");

        $last_migration = $this->getLastMigration();
        // 日付順にファイルをソート
        $all_migrations = $this->getAllMigrationFiles();
        $start_index = ($last_migration) ? array_search($last_migration, $all_migrations) + 1 : 0;

        for ($i = $start_index; $i < count($all_migrations); $i++) {
            $filename = $all_migrations[$i];

            // マイグレーションファイルを読み込む
            include_once ($filename);

            $migration_class = $this->getClassnameFromMigrationFilename($filename);
            $migration = new $migration_class();

            // マイグレーションを実行
            $this->log(sprintf("%sのマイグレーションを実行しています。", $migration_class));
            $queries = $migration->up();
            if (empty($queries)) {
                throw new \Exception("マイグレーションファイルのクエリが空です。");
            }

            // クエリを実行
            $this->processQueries($queries);
            $this->insertMigration($filename);
        }

        $this->log("マイグレーションが完了しました。\n");
    }

    // マイグレーションファイルからクラス名を取得する関数
    private function getClassnameFromMigrationFilename(string $filename): string
    {
        // 正規表現でクラス名を取得
        // / ... 正規表現の開始
        // () ... グループ
        // [] ... 文字クラス
        // ^ ... 特定の文字以外
        // + ... 直前の文字が1文字以上
        // ([^_]+) ... アンダースコア以外の文字が1文字以上
        if (preg_match('/([^_]+)\.php$/', $filename, $matches)) {
            return sprintf("%s\%s", 'Database\Migrations', $matches[1]);
        } else {
            throw new \Exception("クラス名の取得に失敗しました。");
        }
    }

    // 最後に行ったマイグレーションを取得する関数
    private function getLastMigration(): ?string
    {
        $mysqli = new MySQLWrapper();
        $query = "SELECT filename FROM migrations ORDER BY id DESC LIMIT 1";
        $result = $mysqli->query($query);

        // カラムが存在する場合は、ファイル名を返す
        if ($result && $result->num_rows > 0) {
            $row = $result->fetch_assoc();
            return $row['filename'];
        }
        return null;
    }

    // マイグレーションファイルを全て取得する関数
    private function getAllMigrationFiles(string $order = 'asc'): array
    {
        $directory = sprintf("%s/../../Database/Migrations", __DIR__);
        $this->log($directory);

        // glob ... ワイルドカード文字列と一致するファイルを取得
        $all_files = glob($directory . "/*.php");

        // デフォルトは昇順でファイルをソート
        usort($all_files, function ($a, $b) use ($order) {
            $compare_result = strcmp($a, $b);
            return ($order === 'asc') ? $compare_result : -$compare_result;
        });

        return $all_files;
    }

    private function processQueries(array $queries): void
    {
        $mysqli = new MySQLWrapper();

        foreach ($queries as $query) {
            $result = $mysqli->query($query);
            if (!$result) {
                throw new \Exception("クエリの実行に失敗しました。");
            } else {
                $this->log("クエリの実行が完了しました。");
            }
        }
    }

    private function insertMigration(string $filename): void
    {
        $mysqli = new MySQLWrapper();

        $statement = $mysqli->prepare("INSERT INTO migrations (filename) VALUES (?)");
        if (!$statement) {
            throw new \Exception("クエリの準備に失敗しました。");
        }

        // 準備されたクエリに実際のファイル名を挿入
        $statement->bind_param('s', $filename);

        // ステートメントの実行
        if (!$statement->execute()) {
            throw new \Exception("クエリの実行に失敗しました。");
        }

        // ステートメントを閉じる
        $statement->close();
    }

    private function rollback(): void
    {
        $this->log("Rolling back...");
        $this->log("ロールバックが完了しました。\n");
    }
}
```

- `php console migrate --init`
- マイグレーションファイルが実行され、usersテーブルとpostsテーブルが作成された

```bash
{"rollback":false,"init":true}
マイグレーションテーブルを作成します。
マイグレーションテーブルの作成が完了しました。
マイグレーションを開始します。
Migrating...
/home/vboxuser/dev/pj5-blog/Commands/Programs/../../Database/Migrations
Database\Migrations\CreateUserTable1のマイグレーションを実行しています。
クエリの実行が完了しました。
Database\Migrations\CreatePostTable1のマイグレーションを実行しています。
クエリの実行が完了しました。
マイグレーションが完了しました。
```

- rollbackオプションの処理を追加

```php
    private function rollback(int $n = 1): void
    {
        $this->log("Rolling back {$n} migration(s)...");

        $last_migration = $this->getLastMigration();
        $all_migrations = $this->getAllMigrationFiles();

        $last_migration_index = array_search($last_migration, $all_migrations);

        if (!$last_migration_index) {
            $this->log("最後に実行したマイグレーションが見つかりませんでした。");
            return;
        }

        $count = 0;
        // 最後に実行したマイグレーションからn個分のマイグレーションをロールバック
        for ($i = $last_migration_index; $count < $n; $i--) {
            $filename = $all_migrations[$i];
            $this->log("Rolling back {$filename}...");

            include_once ($filename);

            $migration_class = $this->getClassnameFromMigrationFilename($filename);
            $migration = new $migration_class();

            $queries = $migration->down();
            if (empty($queries)) {
                throw new \Exception("マイグレーションファイルのクエリが空です。");
            }

            $this->processQueries($queries);
            $this->removeMigration($filename);
            $count++;
        }

        $this->log("ロールバックが完了しました。\n");
    }

    private function removeMigration(string $filename): void
    {
        $mysqli = new MySQLWrapper();
        $statement = $mysqli->prepare("DELETE FROM migrations WHERE filename = ?");

        if (!$statement) {
            throw new \Exception("クエリの準備に失敗しました。(" . $mysqli->errno . ")" . $mysqli->error);
        }

        $statement->bind_param('s', $filename);
        if (!$statement->execute()) {
            throw new \Exception("クエリの実行に失敗しました。(" . $mysqli->errno . ")" . $mysqli->error);
        }

        $statement->close();
    }

```

- `php console migrate --rollback n`
- テーブルのロールバックがｎ回行われた

マイグレーションベースのメリット… どのような段階でもスキーママイグレーションを同期できるため、ローカル開発環境や事前デプロイ時に新しいデータを追加しない状態のままロールバックができる

デメリット… 本番サーバでのロールバックはデータ損失のリスクがあるので注意

→本番環境では元に戻す新しいマイグレーションを作成するのが一般的

---

マイグレーションベース

課題①：BlogBook

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%205.png)

- `php console code-gen migration --name Create...Table`
- 定義に沿って各ファイルを編集
- `php console migrate --init` テーブル作成

*複数のカラムの組み合わせを一意の主キーとする場合は、複合主キーとして書く

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreateCommentLikeTable implements SchemaMigration
{
    public function up(): array
    {
        // マイグレーション処理を書く
        return [
            "CREATE TABLE comment_likes(
                user_id BIGINT,
                comment_id BIGINT,
                PRIMARY KEY (user_id, comment_id), //
                FOREIGN KEY (user_id) REFERENCES users(id) ON DELETE CASCADE,
                FOREIGN KEY (comment_id) REFERENCES comments(id) ON DELETE CASCADE
            )"
        ];
    }

    public function down(): array
    {
        // ロールバック処理を書く
        return [
            "DROP TABLE comment_likes"
        ];
    }
}
```

課題②：BlogBook　拡張

初版のスキーマが完成し、本番環境に展開した後、BlogBook のスキーマはいくつかの改良が必要です。初版のスキーマが既に本番環境にデプロイされているため、新たなマイグレーションを追加して変更とロールバックを容易にします。BlogBooks の第二版を実現するためにテーブルに順次小規模な変更を施してください。

up メソッドで加える各変更には、down メソッドによる対応する逆の操作が必要です。例えば、up メソッドで列を削除する場合、down メソッドではその列を再び追加する処理を行います。

このセクションに取り掛かる前に、初版のスキーマのバックアップを作成し、その時点でのデータを保持しておいてください。

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%206.png)

- tag, categoryの代わりにtaxonomy(分類)とtaxonomy_termテーブルを作成し、postの分類に使う
- subscription(会員の登録状況)をuserから分離してテーブルを追加
- →subscriptionテーブルのup()にusersテーブルのsubscription列削除の処理を追加し、down()で逆に列を追加させるようにする
- taxonomyテーブルのup()でcategory, tag, post_tagテーブルを削除、down()で作成

*SQL文法でやりがちなこと

- CREATE TABLE 文の最後の要素の後にカンマつけて文法エラー

---

状態ベースのスキーマ管理

- マイグレーションベース….段階的にスキーマに変更を加える（一般に使われる方法）
- 状態ベース….希望する状態のスキーマをあらかじめ定義し、現在のテーブルの状態と比較して、必要に応じてツールによって自動でマイグレーションファイルを生成させる
- （DBをリセットできる状況や、大規模な開発でより有効）

自動でスキーマを管理するようにあらかじめシステムを組むため、最初の開発・設計がより複雑

テンプレ

```php
type ForeignKeyDefinition = {
  'referenceTable' => string,
  'referenceColumn' => string, 
  ?'onDelete' => string, // オプション
};

type ColumnDefinition = {
  'dataType' => string,      
  ?'constraints' => string, // オプション
  ?'primaryKey' => bool, // オプション
  'nullable' => bool,     
  ?'foreignKey' => ForeignKeyDefinition, // オプション
  // ...他の列のプロパティも許可されています
};

type TableDefinition = {
  string => ColumnDefinition // キーとして列名
};

type DatabaseSchemaDefinition = dict<string, TableDefinition>; // キーとしてテーブル名

```

Database/setup.php

```php
<?php

return [
    'User' => [
        'userID' => [
            'dataType' => 'INT',
            'constraints' => 'AUTO_INCREMENT',
            'primaryKey' => true,
            'nullable' => false,
        ],
        'username' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => false,
        ],
        'email' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => false,
        ],
        'password' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => false,
        ],
        'email_confirmed_at' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => true,
        ],
        'created_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ],
        'updated_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ]
    ],
    'Post' => [
        'postID' => [
            'dataType' => 'INT',
            'constraints' => 'AUTO_INCREMENT',
            'primaryKey' => true,
            'nullable' => false,
        ],
        'title' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => false,
        ],
        'content' => [
            'dataType' => 'TEXT',
            'nullable' => false,
        ],
        'created_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ],
        'updated_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ],
        'userID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'User',
                'referenceColumn' => 'userID',
                'onDelete' => 'CASCADE'
            ],
            'nullable' => false,
        ]
    ],
    'Comment' => [
        'commentID' => [
            'dataType' => 'INT',
            'constraints' => 'AUTO_INCREMENT',
            'primaryKey' => true,
            'nullable' => false,
        ],
        'commentText' => [
            'dataType' => 'VARCHAR(255)',
            'nullable' => false,
        ],
        'created_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ],
        'updated_at' => [
            'dataType' => 'DATETIME',
            'nullable' => false,
        ],
        'userID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'User',
                'referenceColumn' => 'userID',
                'onDelete' => 'CASCADE'
            ],
            'nullable' => false,
        ],
        'postID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'Post',
                'referenceColumn' => 'postID',
                'onDelete' => 'CASCADE'
            ],
            'nullable' => false,
        ]
    ],
    'PostLike' => [
        'userID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'User',
                'referenceColumn' => 'userID',
                'onDelete' => 'CASCADE'
            ],
            'primaryKey' => true,
            'nullable' => false,
        ],
        'postID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'Post',
                'referenceColumn' => 'postID',
                'onDelete' => 'CASCADE'
            ],
            'primaryKey' => true,
            'nullable' => false,
        ]
    ],
    'CommentLike' => [
        'userID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'User',
                'referenceColumn' => 'userID',
                'onDelete' => 'CASCADE'
            ],
            'primaryKey' => true,
            'nullable' => false,
        ],
        'commentID' => [
            'dataType' => 'INT',
            'foreignKey' => [
                'referenceTable' => 'Comment',
                'referenceColumn' => 'commentID',
                'onDelete' => 'CASCADE'
            ],
            'primaryKey' => true,
            'nullable' => false,
        ]
    ]
];

```

Commands/Programs/StateMigrate.php

```php
<?php

namespace Commands\Programs;

use Commands\AbstractCommand;
use Commands\Argument;
use Database\MySQLWrapper;

class StateMigrate extends AbstractCommand
{
    protected static ?string $alias = 'state-migrate';

    public static function getArgs(): array
    {
        return [
            (new Argument('init'))->description('Table Initialization')->required(true),
        ];
    }

    public function execute(): int
    {
        $this->log("Starting state migration...");

        // データベース全体をクリーンアップします
        $this->cleanDatabase();
        
        $desiredSchema = include('./Database/state.php');  
        foreach ($desiredSchema as $table => $columns) {
            $this->stateToSchema($table, $columns);
        }

        $this->log("State migration completed.");
        return 0;
    }

    private function cleanDatabase(): void
    {
        $mysqli = new MySQLWrapper();

        // ドロップ中のエラーを防ぐため、外部キーチェックを無効化
        // (外部キーを持つテーブルを消したあとで関連するテーブルを消す処理に入るときにキーを参照しようとしないため）
        $mysqli->query("SET foreign_key_checks = 0");

        $result = $mysqli->query("SHOW TABLES");
        while ($row = $result->fetch_row()) {
            $table = $row[0];
            $this->log("Dropping table $table");
            $mysqli->query("DROP TABLE `$table`");
        }

        // 外部キーチェックを最有効化
        $mysqli->query("SET foreign_key_checks = 1");
    }
    
    private function stateToSchema(string $table, array $columns): void
    {
        $mysqli = new MySQLWrapper();

        $columnDefinitions = [];
        $keys = [];
        $primaryKeysColumns = [];
        foreach ($columns as $columnName => $columnProps) {
            $definition = "`$columnName` {$columnProps['dataType']}";

            if (isset($columnProps['constraints'])) {
                $definition .= " {$columnProps['constraints']}";
            }

            if (isset($columnProps['nullable']) && !$columnProps['nullable']) {
                $definition .= " NOT NULL";
            }

            if (isset($columnProps['primaryKey']) && $columnProps['primaryKey']) {
               
	    $primaryKeysColumns[] = $columnName;
            }

            if (isset($columnProps['foreignKey'])) {
                $fk = $columnProps['foreignKey'];
                $onDelete = isset($fk['onDelete']) ? "ON DELETE {$fk['onDelete']}" : "";
                $keys[] = "FOREIGN KEY (`$columnName`) REFERENCES {$fk['referenceTable']}({$fk['referenceColumn']}) $onDelete";
            }

            $columnDefinitions[] = $definition;
        }

        if (count($primaryKeysColumns) > 0) {
            $keys[] = "PRIMARY KEY (" . implode(", ", $primaryKeysColumns) . ")";
        }

        $columnSQL = implode(', ', $columnDefinitions);
        $keysSQL = implode(', ', $keys);

        $createTableQuery = "CREATE TABLE IF NOT EXISTS `$table` ($columnSQL, $keysSQL)";

        $result = $mysqli->query($createTableQuery);
        if ($result === false) throw new \Exception("Failed to ensure table $table matches desired state.");
        else $this->log("Ensured table $table matches desired state.");
    }
}

```

- registry.phpにStateMigrateコマンドを追加
- `php console state-migrate --init`

---

DML操作

データ挿入, 読み取り、更新、削除（CRUD）

sql-test.php

```php
<?php

use Database\MySQLWrapper;

spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

$mysqli = new MySQLWrapper();

$createTableQuery = "
    CREATE TABLE IF NOT EXISTS students (
      id INT PRIMARY KEY AUTO_INCREMENT,
      name VARCHAR(100),
      age INT,
      major VARCHAR(50)
    )
";

$mysqli->query($createTableQuery);

// データの作成（create）
$students_data = [
    ['John', 18, 'Mathematics'],
    ['Jane', 19, 'Physics'],
    ['Doe', 20, 'Chemistry'],
    ['Smith', 21, 'Biology'],
    ['Brown', 22, 'Computer Science'],
];

foreach ($students_data as $student) {
    $insert_query = "
        INSERT INTO students (name, age, major)
        VALUES ('{$student[0]}', {$student[1]}, '{$student[2]}')
    ";
    $mysqli->query($insert_query);
}

// データの読み取り（read）   
$select_query = "SELECT * FROM students";
$result = $mysqli->query($select_query);

// 一行ずつ取得
while ($row = $result->fetch_assoc()) {
    echo "ID: " . $row['id'] . ", Name: " . $row['name'] . ", Age: " . $row['age'] . ", Major: " . $row['major'] . PHP_EOL;
}

echo "--------------------------------------" . PHP_EOL;
echo "データの更新" . PHP_EOL;

// データの更新（update）
$updates = [
    ['John', 'Philosophy'],
    ['Jane', 'Astronomy'],
    ['Doe', 'Geology'],
    ['Smith', 'Physics'],
    ['Brown', 'Computer Engineering'],
];

foreach ($updates as $update) {
    $update_query = "
        UPDATE students
        SET major = '{$update[1]}'
        WHERE name = '{$update[0]}'
    ";
    $mysqli->query($update_query);
}

$select_query = "SELECT * FROM students";
$result = $mysqli->query($select_query);

// 一行ずつ取得
while ($row = $result->fetch_assoc()) {
    echo "ID: " . $row['id'] . ", Name: " . $row['name'] . ", Age: " . $row['age'] . ", Major: " . $row['major'] . PHP_EOL;
}

echo "--------------------------------------" . PHP_EOL;
echo "データの削除" . PHP_EOL;

// データの削除（delete）
$students_to_delete = ['John', 'Jane'];

foreach ($students_to_delete as $student) {
    $delete_query = "
        DELETE FROM students
        WHERE name = '{$student}'";
    $mysqli->query($delete_query);
}

$select_query = "SELECT * FROM students";
$result = $mysqli->query($select_query);

// 一行ずつ取得
while ($row = $result->fetch_assoc()) {
    echo "ID: " . $row['id'] . ", Name: " . $row['name'] . ", Age: " . $row['age'] . ", Major: " . $row['major'] . PHP_EOL;
}
```

---

セキュリティ

疑似的なSQLインジェクションを起こしてみる

php console code-gen migration --name CreateUsersTestTable1

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreateUsersTestTable1 implements SchemaMigration
{
    public function up(): array
    {
        return [
            "CREATE TABLE test_users (
                id BIGINT PRIMARY KEY AUTO_INCREMENT,
                username VARCHAR(255) NOT NULL,
                email VARCHAR(255) NOT NULL UNIQUE,
                password VARCHAR(255) NOT NULL,
                role VARCHAR(255)
            )"
        ];
    }

    public function down(): array
    {
        return [
            "DROP TABLE test_users"
        ];
    }
}

```

- データの挿入

insert_user_sample.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();

$charset = $mysqli->get_charset();
if ($charset === null)
    throw new Exception('Charset could be read');

$users_data = [
    ['admin', 'admin@example.com', '32u08fa9', 'admin'],
    ['user_1', 'user-1@example.com', 'jd9aewh3', 'user'],
    ['user_2', 'user-2@example.com', '9afeovz', 'user']
];

foreach ($users_data as $user) {
    $insert_query = "INSERT INTO test_users (username, email, password, role) VALUES ('$user[0]', '$user[1]', '$user[2]', '$user[3]')";
    $mysqli->query($insert_query);
}

```

- ログインページ

public/test_login.html

```html
<html lang="ja">

<head>
    <meta charset="UTF-8">
    <title>ログイン</title>
</head>

<body>
    <form action="/test_login.php" method="post">
        <label for="username">ユーザー名:</label>
        <input type="text" id="username" name="username" required>
        <label for="password">パスワード:</label>
        <input type="password" id="password" name="password" required>
        <input type="submit" value="ログイン">
    </form>
</body>

</html>
```

- インジェクション対策されてないtest_login.php

public/test_login.php

```php
<?php
// ファイルを探すためのパスを設定
echo get_include_path();
set_include_path(get_include_path() . PATH_SEPARATOR . realpath(__DIR__ . '/..'));
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

use Database\MySQLWrapper;

$mysqli = new MySQLWrapper();
$charset = $mysqli->get_charset();
if ($charset === null)
    throw new Exception('Charset could be read');

// フォームから送信されたユーザー名とパスワードを取得
$username = $_POST['username'] ?? '';
$password = $_POST['password'] ?? '';

$query = "SELECT * FROM test_users WHERE username = '$username' AND password = '$password';";
$result = $mysqli->query($query);
$user_data = $result->fetch_assoc();

if ($user_data) {
    $login_username = $userData["username"];
    $login_email = $userData["email"];
    $login_role = $userData["role"];

    echo "ログイン成功<br/>";
    echo "こんにちは、$login_username<br/>";
    if ($login_role == 'admin') {
        echo "role: admin でログインしています。<br/>";
        echo "password: $password<br/>";
    }
} else {
    echo "ログイン失敗<br/>";
}

```

- `cd public`
- `php -S localhost:8000`を立ち上げ、http://127.0.0.1:8000/test_login.htmlからログイン画面にアクセス
- user1でログイン（パスワード：jd9aewh3）

↑をしたときにDB側でおこっている処理

```sql
SELECT * FROM users WHERE username = 'user1' AND password = 'jd9aewh3';
```

SQLインジェクション攻撃が出来てしまう

- ユーザー名を`admin' ; --` としてログイン　パスワードなしでログインできてしまう
- ;でSQL文がAND直前で終わり、AND以降がコメントアウトされてしまうから

裏側

```sql
SELECT * FROM users WHERE username = 'admin' ; -- ' AND password = '';

```

↓のようにSQLが改ざんされてしまう

```sql
SELECT * FROM users WHERE username = 'admin' ;

```

代表的な攻撃

- クラシックSQLインジェクション… 上記のような例
- ブラインドSQLインジェクション… DBにSQLに関する質問を繰り返し、その応答の違いから判断して情報を盗む
- タイムベースブラインドSQLインジェクション… SLEEP(time)やBENCHMARK(count, expr)など特定のキーワードを用いて、ページ応答までの時間から脆弱性があるかどうか判断する

インジェクションを防ぐmysqliの機能

- prepare() … 実際のデータの代わりにプレースホルダーを使用して、完全なクエリを最初に書く関数(SQL文のみを渡す）

```php
<?php

$name = "Alice";
$stmt = $mysqli->prepare("SELECT * FROM students WHERE name = ?");
$stmt->bind_param("s", $name);
$stmt->execute();
$result = $stmt->get_result();
```

ユーザーからの入力は常にデータとして扱われ、実行可能なコードとしては扱われない

代表的なセキュリティ対策

- 入力のサニタイズ… ユーザーからの入力データはそのまま信用せず、コードに組み込む前に検証する（htmlspecialchars(), strip_tags()ではHTMLタグや特殊文字を無効化できる）
- 厳格な型付け（年齢フィールドは数字以外の入力ははじくなど）
- ユーザーロールごとに権限づけ、機能制限
- セッション管理
- エンドユーザーに対して詳細すぎるエラーメッセージは表示しない
- DBへのアクセス権限を最小限化
- 定期的にバックアップ
- HTTPSなどの安全な通信セキュリティ確保
- セキュリティ設定や処理を定期的にレビュー

---

テーブルの初期データ投入（コンピュータ部品情報のAPI作成）

DBにアクセスするアプリ開発の際、一般的には初期データ（ダミーデータ）を前もって準備して設定しておく

エンドポイント

- /random/part
- /parts?id={id}
- /types?type={type}&page={page}&perpage={count}
- /random/computer

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%207.png)

- `php console code-gen migration --name CreateComputerPartsTable1`

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreateComputerPartsTable1 implements SchemaMigration
{
    public function up(): array
    {
        return [
            "CREATE TABLE computer_parts (
                id INT PRIMARY KEY AUTO_INCREMENT,
                name VARCHAR(255) NOT NULL,
                type VARCHAR(50) NOT NULL,
                brand VARCHAR(255) NOT NULL,
                model_number VARCHAR(100),
                release_date DATE,
                description TEXT,
                performance_score INT,
                market_price DECIMAL(12, 2),
                rsm DECIMAL(12, 2),
                power_consumptionw FLOAT,
                lengthm DOUBLE,
                widthm DOUBLE,
                heightm DOUBLE,
                lifespan INT,
                created_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP,
                updated_at DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            )"
        ];
    }

    public function down(): array
    {
        return [
            "DROP TABLE computer_parts"
        ];
    }
}

```

- `php console migrate --init`

- 初期データを投入（シーディング）するシステムを作成

Database/Seeder.php

```php
<?php

namespace Database;

interface Seeder
{
    public function seed(): void;

    public function createRowData(): array;
}

```

Database/AbstractSeeder.php

```php
<?php
namespace Database;

use Database\MySQLWrapper;

abstract class AbstractSeeder implements Seeder
{
    protected MySQLWrapper $conn;
    protected ?string $tableName = null;

    // テーブルカラムは、'data_type' と 'column_name' を含む連想配列の配列です。
    protected array $tableColumns = [];

    // キーはデータ型の文字列で、値はbind_paramの文字列
    const AVAILABLE_TYPES = [
        'int' => 'i',
        // PHPのfloatはdouble型の精度
        'float' => 'd',
        'string' => 's',
    ];

    public function __construct(MySQLWrapper $conn)
    {
        $this->conn = $conn;
    }

    public function seed(): void
    {
        // データを作成
        $data = $this->createRowData();

        if ($this->tableName === null)
            throw new \Exception('Class requires a table name');
        if (empty($this->tableColumns))
            throw new \Exception('Class requires a columns');

        foreach ($data as $row) {
            // 行を検証
            $this->validateRow($row);
            $this->insertRow($row);
        }
    }

    // 各行をtableColumnsと照らし合わせて検証する関数
    protected function validateRow(array $row): void
    {
        if (count($row) !== count($this->tableColumns))
            throw new \Exception('Row does not match the ');

        foreach ($row as $i => $value) {
            $columnDataType = $this->tableColumns[$i]['data_type'];
            $columnName = $this->tableColumns[$i]['column_name'];

            if (!isset(static::AVAILABLE_TYPES[$columnDataType]))
                throw new \InvalidArgumentException(sprintf("The data type %s is not an available data type.", $columnDataType));

            // 値のデータタイプを返すget_debug_type()とgettype()がある
            // https://www.php.net/manual/en/function.get-debug-type.php 
            // get_debug_type ... ネイティブPHP8のタイプを返す。 (floatsのgettype()の場合は'float'ではなく、'double'を返す)
            if (get_debug_type($value) !== $columnDataType)
                throw new \InvalidArgumentException(sprintf("Value for %s should be of type %s. Here is the current value: %s", $columnName, $columnDataType, json_encode($value)));
        }
    }

    // 各行を挿入する関数
    protected function insertRow(array $row): void
    {
        // カラム名を取得
        $columnNames = array_map(function ($columnInfo) {
            return $columnInfo['column_name'];
        }, $this->tableColumns);

        // プレースホルダーの?はcount($row) - 1回繰り返され、最後の?の後にはカンマをつけない
        // そこにbind_paramで値を挿入する
        $placeholders = str_repeat('?,', count($row) - 1) . '?';

        $sql = sprintf(
            'INSERT INTO %s (%s) VALUES (%s)',
            $this->tableName,
            implode(', ', $columnNames),
            $placeholders
        );

        // prepare()はSQLステートメントを準備し、ステートメントオブジェクトを返す
        $stmt = $this->conn->prepare($sql);

        // データ型配列を結合して文字列にする
        $dataTypes = implode(array_map(function ($columnInfo) {
            return static::AVAILABLE_TYPES[$columnInfo['data_type']];
        }, $this->tableColumns));

        // 文字の配列（文字列）を取り、それぞれに行の値を挿入する
        // 例：$stmt->bind_param('iss', ...array_values([1, 'John', 'john@example.com'])) は、ステートメントに整数(i)、文字列(s)、文字列(s)を挿入する
        $row_values = array_values($row);
        $stmt->bind_param($dataTypes, ...$row_values);

        $stmt->execute();
    }
}
```

Database/Seeds/ComputerPartsSeeder.php

```php
<?php

namespace Database\Seeds;

use Database\AbstractSeeder;

class ComputerPartsSeeder extends AbstractSeeder
{
    protected ?string $tableName = 'computer_parts';
    protected array $tableColumns = [
        [
            'data_type' => 'string',
            'column_name' => 'name'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'type'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'brand'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'model_number'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'release_date'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'description'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'performance_score'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'market_price'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'rsm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'power_consumptionw'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'lengthm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'widthm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'heightm'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'lifespan'
        ]
    ];

    public function createRowData(): array
    {
        return [
            [
                'Ryzen 9 5900X',
                'CPU',
                'AMD',
                '100-000000061',
                '2020-11-05',
                'A high-performance 12-core processor.',
                90,
                549.99,
                0.05,
                105.0,
                0.04,
                0.04,
                0.005,
                5
            ],
            [
                'GeForce RTX 3080',
                'GPU',
                'NVIDIA',
                '10G-P5-3897-KR',
                '2020-09-17',
                'A powerful gaming GPU with ray tracing support.',
                93,
                699.99,
                0.04,
                320.0,
                0.285,
                0.112,
                0.05,
                5
            ],
            [
                'Samsung 970 EVO SSD',
                'SSD',
                'Samsung',
                'MZ-V7E500BW',
                '2018-04-24',
                'A fast NVMe M.2 SSD with 500GB storage.',
                88,
                79.99,
                0.02,
                5.7,
                0.08,
                0.022,
                0.0023,
                5
            ],
            [
                'Corsair Vengeance LPX 16GB',
                'RAM',
                'Corsair',
                'CMK16GX4M2B3200C16',
                '2015-08-10',
                'A DDR4 memory kit operating at 3200MHz.',
                85,
                69.99,
                0.03,
                1.2,
                0.137,
                0.03,
                0.007,
                7
            ],
            [
                'ASUS ROG Strix B550-F',
                'Motherboard',
                'ASUS',
                '90MB14F0-M0EAY0',
                '2020-06-16',
                'A high-end motherboard with PCIe 4.0 support.',
                87,
                189.99,
                0.03,
                0.0,
                0.305,
                0.244,
                0.005,
                5
            ],
            [
                'EVGA SuperNOVA 750 G5',
                'PSU',
                'EVGA',
                '220-G5-0750-X1',
                '2019-06-05',
                'A 750W power supply with 80 Plus Gold certification.',
                90,
                129.99,
                0.03,
                750.0,
                0.15,
                0.15,
                0.085,
                7
            ],
            [
                'NZXT H510',
                'Case',
                'NZXT',
                'CA-H510B-W1',
                '2019-08-01',
                'A compact ATX case with a tempered glass side panel.',
                85,
                69.99,
                0.03,
                0.0,
                0.428,
                0.210,
                0.460,
                5
            ]
        ];
    }
}

```

- seedコマンド

Commands/Programs/Seed.php

```php
<?php

namespace Commands\Programs;

use Commands\AbstractCommand;
use Database\MySQLWrapper;
use Database\Seeder;

class Seed extends AbstractCommand
{
    // コマンド名を設定します
    protected static ?string $alias = 'seed';

    public static function getArgs(): array
    {
        return [];
    }

    public function execute(): int
    {
        $this->runAllSeeds();
        return 0;
    }

    function runAllSeeds(): void
    {
        $directory_path = __DIR__ . '/../../Database/Seeds';

        // Seedsディレクトリ内のファイルを取得
        $files = scandir($directory_path);

        foreach ($files as $file) {
            if (pathinfo($file, PATHINFO_EXTENSION) === 'php') {
                // クラス名をファイル名から取得
                $class_name = 'Database\Seeds\\' . pathinfo($file, PATHINFO_FILENAME);

                // シードファイルを読み込む
                include_once $directory_path . '/' . $file;

                if (class_exists($class_name) && is_subclass_of($class_name, Seeder::class)) {
                    $seeder = new $class_name(new MySQLWrapper());
                    $seeder->seed();
                } else
                    throw new \Exception('Seeder must be a class that subclasses the seeder interface');
            }
        }
    }
}
```

- `php console seed` 

＊class_exists()がfalse　→ファイル名が違っていたため（注意）

computer_partsテーブルにデータが挿入された

---

課題①: computer_partsテーブル　createRowData()のリファクタリング

- fakerライブラリでデータをランダム生成できるようにする
- １万行のデータをDBに追加できるかどうかテスト

- Composerを使ってfakerをインストール

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

Database/ComputerPartsSeeder.php

```php
<?php

namespace Database\Seeds;

require_once 'vendor/autoload.php';

use Database\AbstractSeeder;

class ComputerPartsSeeder extends AbstractSeeder
{
    protected ?string $tableName = 'computer_parts';
    protected array $tableColumns = [
        [
            'data_type' => 'string',
            'column_name' => 'name'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'type'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'brand'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'model_number'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'release_date'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'description'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'performance_score'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'market_price'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'rsm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'power_consumptionw'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'lengthm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'widthm'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'heightm'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'lifespan'
        ]
    ];

    public function createRowData(): array
    {
        return [
            [
                'Ryzen 9 5900X',
                'CPU',
                'AMD',
                '100-000000061',
                '2020-11-05',
                'A high-performance 12-core processor.',
                90,
                549.99,
                0.05,
                105.0,
                0.04,
                0.04,
                0.005,
                5
            ],
            [
                'GeForce RTX 3080',
                'GPU',
                'NVIDIA',
                '10G-P5-3897-KR',
                '2020-09-17',
                'A powerful gaming GPU with ray tracing support.',
                93,
                699.99,
                0.04,
                320.0,
                0.285,
                0.112,
                0.05,
                5
            ],
            [
                'Samsung 970 EVO SSD',
                'SSD',
                'Samsung',
                'MZ-V7E500BW',
                '2018-04-24',
                'A fast NVMe M.2 SSD with 500GB storage.',
                88,
                79.99,
                0.02,
                5.7,
                0.08,
                0.022,
                0.0023,
                5
            ],
            [
                'Corsair Vengeance LPX 16GB',
                'RAM',
                'Corsair',
                'CMK16GX4M2B3200C16',
                '2015-08-10',
                'A DDR4 memory kit operating at 3200MHz.',
                85,
                69.99,
                0.03,
                1.2,
                0.137,
                0.03,
                0.007,
                7
            ],
            [
                'ASUS ROG Strix B550-F',
                'Motherboard',
                'ASUS',
                '90MB14F0-M0EAY0',
                '2020-06-16',
                'A high-end motherboard with PCIe 4.0 support.',
                87,
                189.99,
                0.03,
                0.0,
                0.305,
                0.244,
                0.005,
                5
            ],
            [
                'EVGA SuperNOVA 750 G5',
                'PSU',
                'EVGA',
                '220-G5-0750-X1',
                '2019-06-05',
                'A 750W power supply with 80 Plus Gold certification.',
                90,
                129.99,
                0.03,
                750.0,
                0.15,
                0.15,
                0.085,
                7
            ],
            [
                'NZXT H510',
                'Case',
                'NZXT',
                'CA-H510B-W1',
                '2019-08-01',
                'A compact ATX case with a tempered glass side panel.',
                85,
                69.99,
                0.03,
                0.0,
                0.428,
                0.210,
                0.460,
                5
            ],
            // fakerのダミーデータを10000件生成
            ...array_map(function () {
                return [
                    \Faker\Factory::create()->name,
                    \Faker\Factory::create()->randomElement(['CPU', 'GPU', 'SSD', 'RAM', 'Motherboard', 'PSU', 'Case']),
                    \Faker\Factory::create()->company,
                    \Faker\Factory::create()->bothify('??????????'),
                    \Faker\Factory::create()->date,
                    \Faker\Factory::create()->text,
                    \Faker\Factory::create()->numberBetween(0, 100),
                    \Faker\Factory::create()->randomFloat(2, 0, 1000),
                    \Faker\Factory::create()->randomFloat(2, 0, 1),
                    \Faker\Factory::create()->randomFloat(2, 0, 1000),
                    \Faker\Factory::create()->randomFloat(2, 0, 10),
                    \Faker\Factory::create()->randomFloat(2, 0, 10),
                    \Faker\Factory::create()->randomFloat(2, 0, 10),
                    \Faker\Factory::create()->numberBetween(0, 10)
                ];

            }, range(0, 9999))
        ];
    }
}

```

10000件のコンピュータのダミーデータが挿入された

課題②: cars, car_partsテーブルのシーダーファイルを追加、code-genコマンド拡張

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%208.png)

- テーブルをまだ作ってないのでマイグレーションファイルをいつも通り作成　省略
- それぞれのシーダーファイルを作成　`seed`コマンド

Database/Seeds/CarsSeeder.php

```php
<?php

namespace Database\Seeds;

use Database\AbstractSeeder;

require_once 'vendor/autoload.php';

class CarsSeeder extends AbstractSeeder
{

    protected ?string $tableName = 'cars';

    protected array $tableColumns = [
        [
            "data_type" => "string",
            "column_name" => "name"
        ],
        [
            "data_type" => "string",
            "column_name" => "make"
        ],
        [
            "data_type" => "string",
            "column_name" => "model"
        ],
        [
            "data_type" => "int",
            "column_name" => "year"
        ],
        [
            "data_type" => "string",
            "column_name" => "color"
        ],
        [
            "data_type" => "float",
            "column_name" => "price"
        ],
        [
            "data_type" => "float",
            "column_name" => "mileage"
        ],
        [
            "data_type" => "string",
            "column_name" => "transmission"
        ],
        [
            "data_type" => "string",
            "column_name" => "engine"
        ],
        [
            "data_type" => "string",
            "column_name" => "status"
        ]
    ];

    public function createRowData(): array
    {
        return [
            ...array_map(function () {
                $makers = ['Toyota', 'Honda', 'Nissan', 'Mitsubishi', 'Subaru'];
                $models_map = [
                    'Toyota' => ['Prius', 'Corolla', 'Camry', 'RAV4', 'Highlander'],
                    'Honda' => ['Civic', 'Accord', 'CR-V', 'Pilot', 'Odyssey'],
                    'Nissan' => ['Sentra', 'Altima', 'Maxima', 'Rogue', 'Pathfinder'],
                    'Mitsubishi' => ['Mirage', 'Lancer', 'Outlander', 'Eclipse Cross', 'Pajero'],
                    'Subaru' => ['Impreza', 'Legacy', 'Forester', 'Outback', 'Ascent']
                ];

                return [
                    \Faker\Factory::create()->randomElement($makers),
                    \Faker\Factory::create()->randomElement($models_map[$makers[0]]),
                    \Faker\Factory::create()->year(),
                    \Faker\Factory::create()->colorName(),
                    \Faker\Factory::create()->randomFloat(2, 10000, 50000),
                    \Faker\Factory::create()->randomFloat(2, 0, 200000),
                    \Faker\Factory::create()->randomElement(['Automatic', 'Manual']),
                    \Faker\Factory::create()->randomElement(['V6', 'V8', 'V12']),
                    \Faker\Factory::create()->randomElement(['New', 'Used'])
                ];
            }, range(0, 999))
        ];
    }
}

```

先にcarsテーブルにデータを挿入

`php console seed`

Database/Seeds/CarPartsSeeder.php

```php
<?php

namespace Database\Seeds;

use Database\AbstractSeeder;

require_once 'vendor/autoload.php';

class CarPartsSeeder extends AbstractSeeder
{

    protected ?string $tableName = 'car_parts';

    protected array $tableColumns = [
        [
            'data_type' => 'int',
            'column_name' => 'car_id'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'name'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'description'
        ],
        [
            'data_type' => 'float',
            'column_name' => 'price'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'quantity_in_stock'
        ]
    ];

    public function createRowData(): array
    {
        return [
            ...array_map(function () {
                $parts = ['Tires', 'Suspension', 'Brakes', 'Wheels', 'Exhaust'];
                $descriptions_map = [
                    'Suspension' => 'A system of springs, shock absorbers, and linkages that connect a vehicle to its wheels',
                    'Brakes' => 'A device for slowing or stopping a moving vehicle, typically by applying pressure to the wheels',
                    'Tires' => 'A rubber covering, typically inflated or surrounding an inflated inner tube, placed around a wheel to form a flexible contact with the road',
                    'Wheels' => 'A circular object that revolves on an axle and is fixed below a vehicle or other object to enable it to move easily over the ground',
                    'Exhaust' => 'A pipe or duct that carries away waste gases and air from a combustion engine'
                ];
                $max_car_id = 100;

                return [
                    rand(1, $max_car_id),
                    \Faker\Factory::create()->randomElement($parts),
                    \Faker\Factory::create()->randomElement($descriptions_map),
                    \Faker\Factory::create()->randomFloat(2, 100, 1000),
                    \Faker\Factory::create()->randomNumber(2)
                ];
            }, range(0, 9999))
        ];
    }
}
```

1 ~ 100番目の車に対するパーツをランダムに挿入

`php console seed`

課題③：created_at, updated_atカラムを追加

- CURRENT_TIMESTAMPではなくCarbonライブラリを使用
- Carbon::now()でCarbonインスタンスを返す

- `composer require nesbot/carbon`　インストール
- カラムをマイグレーションファイルに追加して再度`php console migrate` （省略）　型はTIMESTAMP
- AbstractSeederクラスのほうで、created_at, updated_atカラムとそれらに対応するプレースホルダー文字列を追加

Database/AbstractSeeder.php

```php
<?php
namespace Database;

require_once 'vendor/autoload.php';

use Database\MySQLWrapper;
use Carbon\Carbon;

abstract class AbstractSeeder implements Seeder
{
    protected MySQLWrapper $conn;
    protected ?string $tableName = null;

    // テーブルカラムは、'data_type' と 'column_name' を含む連想配列の配列。
    protected array $tableColumns = [];

    // キーはデータ型の文字列で、値はbind_paramの文字列
    const AVAILABLE_TYPES = [
        'int' => 'i',
        // PHPのfloatはdouble型の精度
        'float' => 'd',
        'string' => 's',
    ];

    public function __construct(MySQLWrapper $conn)
    {
        $this->conn = $conn;
    }

    public function seed(): void
    {
        // データを作成
        $data = $this->createRowData();

        if ($this->tableName === null)
            throw new \Exception('Class requires a table name');
        if (empty($this->tableColumns))
            throw new \Exception('Class requires a columns');

        foreach ($data as $row) {
            // 行を検証
            $this->validateRow($row);
            $this->insertRow($row);
        }
    }

    // 各行をtableColumnsと照らし合わせて検証する関数
    protected function validateRow(array $row): void
    {
        echo count($row) . PHP_EOL;
        echo count($this->tableColumns) . PHP_EOL;
        if (count($row) !== count($this->tableColumns))
            throw new \Exception('Row does not match the ' . $this->tableName . ' table columns.');

        foreach ($row as $i => $value) {
            $columnDataType = $this->tableColumns[$i]['data_type'];
            $columnName = $this->tableColumns[$i]['column_name'];

            if (!isset(static::AVAILABLE_TYPES[$columnDataType]))
                throw new \InvalidArgumentException(sprintf("The data type %s is not an available data type.", $columnDataType));

            // 値のデータタイプを返すget_debug_type()とgettype()がある
            // https://www.php.net/manual/en/function.get-debug-type.php 
            // get_debug_type ... ネイティブPHP8のタイプを返す。 (floatsのgettype()の場合は'float'ではなく、'double'を返す)
            if (get_debug_type($value) !== $columnDataType)
                throw new \InvalidArgumentException(sprintf("Value for %s should be of type %s. Here is the current value: %s", $columnName, $columnDataType, json_encode($value)));
        }
    }

    // 各行を挿入する関数
    protected function insertRow(array $row): void
    {
        // カラム名を取得
        $columnNames = array_map(function ($columnInfo) {
            return $columnInfo['column_name'];
        }, $this->tableColumns);
        // created_atとupdated_atカラムを追加
        $columnNames = array_merge($columnNames, ['created_at', 'updated_at']);

        // プレースホルダーの?はcount($row) - 1回繰り返され、最後の?の後にはカンマをつけない
        // そこにbind_paramで値を挿入する
        $placeholders = str_repeat('?,', count($columnNames) - 1) . '?';

        $now = Carbon::now();
        $sql = sprintf(
            'INSERT INTO %s (%s) VALUES (%s)',
            $this->tableName,
            implode(', ', $columnNames),
            $placeholders
        );

        // prepare()はSQLステートメントを準備し、ステートメントオブジェクトを返す
        $stmt = $this->conn->prepare($sql);

        // データ型配列を結合して文字列にする
        $dataTypes = implode(array_map(function ($columnInfo) {
            return static::AVAILABLE_TYPES[$columnInfo['data_type']];
        }, $this->tableColumns));
        // created_atとupdated_atのデータ型を追加
        $dataTypes .= 'ss';

        // 文字の配列（文字列）を取り、それぞれに行の値を挿入する
        // 例：$stmt->bind_param('iss', ...array_values([1, 'John', 'john@example.com'])) は、ステートメントに整数(i)、文字列(s)、文字列(s)を挿入する
        $row_values = array_merge(array_values($row), [$now, $now]);
        $stmt->bind_param($dataTypes, ...$row_values);

        $stmt->execute();
    }
}
```

- php console seed

Carbon::now()のままでも、bind_param()での型をsにすることで、作成時のタイムスタンプが挿入された

---

ルーティング

リクエスト取得先へのエンドポイント

- /random/part … コンピュータパーツをランダムに取得
- /parts?id={id} … 特定のidのパーツを取得
- types?type={type}&page={page}&perpage={count} … ページネーションの特定ページの指定されたタイプのパーツをランダムに取得
- /random/computer … ランダムにパーツを取得して、ランダムなコンピュータデータを生成

Routers/routes.php

```php
<?php

return [
    'random/part' => 'random-part',
    'parts' => 'parts',
];
```

random-partページ

Views/random-part.php

```php
<?php

use Database\MySQLWrapper;

$db = new MySQLWrapper();

try {
    // ユーザー入力は関わらないので、SQLインジェクションの対策は不要だが、一貫性のためにprepareを使う
    // ランダムに1つのパーツを取得
    $stmt = $db->prepare("SELECT * FROM computer_parts ORDER BY RAND() LIMIT 1");
    $stmt->execute();
    $result = $stmt->get_result();
    $part = $result->fetch_assoc();
} catch (Exception $e) {
    die("Error fetching random part: " . $e->getMessage());
}

if (!$part) {
    echo "No part found!";
    exit;
}

// TODO: 上記のロジックをViewから分離

// パーツをカード表示
// htmlspecialchar ... 特殊文字をHTMLエンティティに変換する
?>
<div class="card" style="width: 18rem;">
    <div class="card-body">
        <h5 class="card-title">
            <?= htmlspecialchars($part['name']) ?>
        </h5>
        <h6 class="card-subtitle mb-2 text-muted">
            <?= htmlspecialchars($part['type']) ?> -
            <?= htmlspecialchars($part['brand']) ?>
        </h6>
        <p class="card-text">
            <strong>Model:</strong>
            <?= htmlspecialchars($part['model_number']) ?><br />
            <strong>Release Date:</strong>
            <?= htmlspecialchars($part['release_date']) ?><br />
            <strong>Description:</strong>
            <?= htmlspecialchars($part['description']) ?><br />
            <strong>Performance Score:</strong>
            <?= htmlspecialchars($part['performance_score']) ?><br />
            <strong>Market Price:</strong> $
            <?= htmlspecialchars($part['market_price']) ?><br />
            <strong>RSM:</strong> $
            <?= htmlspecialchars($part['rsm']) ?><br />
            <strong>Power Consumption:</strong>
            <?= htmlspecialchars($part['power_consumptionw']) ?>W<br />
            <strong>Dimensions:</strong>
            <?= htmlspecialchars($part['lengthm']) ?>m x
            <?= htmlspecialchars($part['widthm']) ?>m x
            <?= htmlspecialchars($part['heightm']) ?>m<br />
            <strong>Lifespan:</strong>
            <?= htmlspecialchars($part['lifespan']) ?> years<br />
        </p>
        <p class="card-text"><small class="text-muted">Last updated on
                <?= htmlspecialchars($part['updated_at']) ?>
            </small></p>
    </div>
</div>
```

partページ(特定idのパーツ)

Views/part.php

```php
<?php

use Database\MySQLWrapper;

// computer partのIDを取得・確認
$id = $_GET['id'] ?? null;
if (!$id) {
    die("No ID provided for computer part");
}

// DB接続を初期化
$db = new MySQLWrapper();

try {
    // 特定のidのパーツを取得
    $stmt = $db->prepare("SELECT * FROM computer_parts WHERE id = ?");
    // int型にidをバインド
    $stmt->bind_param('i', $id);
    // ステートメントを実行
    $stmt->execute();
    // 結果を取得し、該当項目を１行取得
    $result = $stmt->get_result();
    $part = $result->fetch_assoc();
} catch (Exception $e) {
    die("Error fetching part: " . $e->getMessage());
}

// TODO: 上記のロジックをViewから分離

?>
<div class="card" style="width: 18rem;">
    <div class="card-body">
        <h5 class="card-title">
            <?= htmlspecialchars($part['name']) ?>
        </h5>
        <h6 class="card-subtitle mb-2 text-muted">
            <?= htmlspecialchars($part['type']) ?> -
            <?= htmlspecialchars($part['brand']) ?>
        </h6>
        <p class="card-text">
            <strong>Model:</strong>
            <?= htmlspecialchars($part['model_number']) ?><br />
            <strong>Release Date:</strong>
            <?= htmlspecialchars($part['release_date']) ?><br />
            <strong>Description:</strong>
            <?= htmlspecialchars($part['description']) ?><br />
            <strong>Performance Score:</strong>
            <?= htmlspecialchars($part['performance_score']) ?><br />
            <strong>Market Price:</strong> $
            <?= htmlspecialchars($part['market_price']) ?><br />
            <strong>RSM:</strong> $
            <?= htmlspecialchars($part['rsm']) ?><br />
            <strong>Power Consumption:</strong>
            <?= htmlspecialchars($part['power_consumptionw']) ?>W<br />
            <strong>Dimensions:</strong>
            <?= htmlspecialchars($part['lengthm']) ?>m x
            <?= htmlspecialchars($part['widthm']) ?>m x
            <?= htmlspecialchars($part['heightm']) ?>m<br />
            <strong>Lifespan:</strong>
            <?= htmlspecialchars($part['lifespan']) ?> years<br />
        </p>
        <p class="card-text"><small class="text-muted">Last updated on
                <?= htmlspecialchars($part['updated_at']) ?>
            </small></p>
    </div>
</div>
```

ヘッダーレイアウト

```php
<!doctype html>
<html lang="en">
<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet" crossorigin="anonymous">

    <title>My Computer Parts Store</title>
</head>
<body>
<main class="container mt-5 mb-5">

```

フッターレイアウト

```php
</main> <!-- end of content -->

<footer class="bg-light text-center text-lg-start">
    <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2);">
        © <?= date('Y') ?>:
        <a class="text-dark" href="/">MyComputerPartsStore.com</a>
    </div>
</footer>

</body>
</html>

```

index.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

// ルートをロード
$routes = include ('Routing/routes.php');

// リクエストURIからパスを取得
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$path = ltrim($path, '/');

// ルートパスの一致を確認
if (isset($routes[$path])) {
    $view = $routes[$path];
    $viewPath = sprintf("%s/Views/%s.php", __DIR__, $view);

    if (file_exists($viewPath)) {
        // ヘッダーを設定
        include 'Views/layout/header.php';
        include $viewPath;
        include 'Views/layout/footer.php';
    } else {
        http_response_code(500);
        printf("<br>debug info:<br>%s<br>%s", "Internal error, please contact the admin.");
    }
} else {
    // 一致するルートがない場合、404エラー
    http_response_code(404);
    echo "404 Not Found: The requested route was not found on this server.";
    printf("<br>debug info:<br>%s<br>%s", json_encode($routes), $path);
}

```

- indexを実行　`php -S [localhost:8000](http://localhost:8000) index.php`
- [http://127.0.0.1:8000/random/part](http://127.0.0.1:8000/random/part)  ⇒ ランダムにコンピュータのパーツが１件表示される

---

クライアントサーバでのレンダリング

- SSR（今まで）… サーバ側でコンテンツをすべて作成して、最終的なHTMLをクライアントに返す (例：pars.phpはサーバ側で処理され、クライアントに送り返す最終的なHTMLを作成する）
- CSR … 表示するコンテンツや表示方法をクライアント側での処理により決定する　サーバ側で生成したコンテンツがあれば必要に応じてサーバにリクエストを送って取得する

Response/HTTPRenderer.php

```php
<?php

namespace Response;

interface HTTPRenderer {
    public function getFields(): array;
    public function getContent(): string;
}

```

HTMLのサーバサイドレンダラークラス

Response/Render/HTMLRenderer.php

```php
<?php

namespace Response\Render;

use Response\HTTPRenderer;

class HTMLRenderer implements HTTPRenderer
{
    private string $viewFile;
    private array $data;

    public function __construct(string $viewFile, array $data = [])
    {
        $this->viewFile = $viewFile;
        $this->data = $data;
    }

    // レスポンスヘッダーを返す関数
    public function getFields(): array
    {
        return [
            'Content-Type' => 'text/html; charset=UTF-8',
        ];
    }

    // レスポンスボディを返す関数
    public function getContent(): string
    {
        $viewPath = $this->getViewPath($this->viewFile);

        if (!file_exists($viewPath)) {
            throw new \Exception("View file {$viewPath} does not exist.");
        }

        // ob_startはすべての出力をバッファに取り込みます。
        // このバッファはob_get_cleanによって取得することができ、バッファの内容を返し、バッファをクリアします。
        ob_start();
        // extract関数は、キーを変数として現在のシンボルテーブルにインポートします
        extract($this->data);
        require $viewPath;
        return $this->getHeader() . ob_get_clean() . $this->getFooter();
    }

    // ヘッダーレイアウトを取得する関数
    private function getHeader(): string
    {
        ob_start();
        require $this->getViewPath('layout/header');
        return ob_get_clean();
    }

    // フッターレイアウトを取得する関数
    private function getFooter(): string
    {
        ob_start();
        require $this->getViewPath('layout/footer');
        return ob_get_clean();
    }

    // ビューファイルのパスを取得する関数
    private function getViewPath(string $path): string
    {
        return sprintf("%s/%s/Views/%s.php", __DIR__, '../..', $path);
    }
}

```

JSONレスポンスのレンダラークラス

Response/Render/JSONRenderer.php

```php
<?php

namespace Response\Render;

use Response\HTTPRenderer;

class JSONRenderer implements HTTPRenderer {
    private array $data;

    public function __construct(array $data) {
        $this->data = $data;
    }

    public function getFields(): array {
        return [
            'Content-Type' => 'application/json; charset=UTF-8',
        ];
    }

    public function getContent(): string {
        return json_encode($this->data, JSON_THROW_ON_ERROR);
    }
}

```

- ルートパスをキーとしてHTTPRenderを返すコールバック関数を返すようにする
- MVC
- コールバック関数（Controller）を実行 ⇒ DB（Model）からデータ取得 ⇒ ユーザからの入力を検証 ⇒ HTTPRenderer（View）にデータを渡す

Routing/routes.php

```php
<?php

use Helpers\DatabaseHelper;
use Helpers\ValidationHelper;
use Response\HTTPRenderer;
use Response\Render\HTMLRenderer;
use Response\Render\JSONRenderer;

return [
    'random/part'=>function(): HTTPRenderer{
        $part = DatabaseHelper::getRandomComputerPart();

        return new HTMLRenderer('component/random-part', ['part'=>$part]);
    },
    'parts'=>function(): HTTPRenderer{
        // IDの検証
        $id = ValidationHelper::integer($_GET['id']??null);

        $part = DatabaseHelper::getComputerPartById($id);
        return new HTMLRenderer('component/parts', ['part'=>$part]);
    },
    'api/random/part'=>function(): HTTPRenderer{
        $part = DatabaseHelper::getRandomComputerPart();
        return new JSONRenderer(['part'=>$part]);
    },
    'api/parts'=>function(){
        $id = ValidationHelper::integer($_GET['id']??null);
        $part = DatabaseHelper::getComputerPartById($id);
        return new JSONRenderer(['part'=>$part]);
    },
];

```

- index.phpでrendererを呼び出す

index.php

```php
<?php
spl_autoload_extensions(".php");
spl_autoload_register(function ($class) {
    $namespace = explode('\\', $class);
    $file = __DIR__ . '/' . implode('/', $namespace) . '.php';
    if (file_exists($file)) {
        require_once $file;
    }
});

// デバッグモードを設定
$DEBUG = true;

// ルートをロード
$routes = include ('Routing/routes.php');

// リクエストURIからパスを取得
$path = parse_url($_SERVER['REQUEST_URI'], PHP_URL_PATH);
$path = ltrim($path, '/');

// ルートパスの一致を確認
if (isset($routes[$path])) {
    // コールバックを呼び出してrendererを作成。
    $renderer = $routes[$path]();

    try {
        // ヘッダーを設定
        foreach ($renderer->getFields() as $name => $value) {
            // ヘッダーに設定する値をサニタイズ
            $sanitized_value = filter_var($value, FILTER_SANITIZE_STRING, FILTER_FLAG_NO_ENCODE_QUOTES);

            // サニタイズされた値が元の値と一致する場合、ヘッダーを設定する
            if ($sanitized_value && $sanitized_value === $value) {
                header("{$name}: {$sanitized_value}");
            } else {
                // ヘッダー設定に失敗した場合、ログに記録するか処理する
                // エラー処理によっては、例外をスローするか、デフォルトのまま続行できる
                http_response_code(500);
                if ($DEBUG)
                    print ("Failed setting header - original: '$value', sanitized: '$sanitized_value'");
                exit;
            }

            print ($renderer->getContent());
        }
    } catch (Exception $e) {
        http_response_code(500);
        print ("Internal error, please contact the admin.<br>");
        if ($DEBUG)
            print ($e->getMessage());
    }
} else {
    // 一致するルートがない場合、404エラー
    http_response_code(404);
    echo "404 Not Found: The requested route was not found on this server.";
}

```

DBデータ取得用のヘルパークラス

Helpers/DatabaseHelper.php

```php
<?php

namespace Helpers;

use Database\MySQLWrapper;
use Exception;

class DatabaseHelper
{
    public static function getRandomComputerPart(): array{
        $db = new MySQLWrapper();

        $stmt = $db->prepare("SELECT * FROM computer_parts ORDER BY RAND() LIMIT 1");
        $stmt->execute();
        $result = $stmt->get_result();
        $part = $result->fetch_assoc();

        if (!$part) throw new Exception('Could not find a single part in database');

        return $part;
    }

    public static function getComputerPartById(int $id): array{
        $db = new MySQLWrapper();

        $stmt = $db->prepare("SELECT * FROM computer_parts WHERE id = ?");
        $stmt->bind_param('i', $id);
        $stmt->execute();

        $result = $stmt->get_result();
        $part = $result->fetch_assoc();

        if (!$part) throw new Exception('Could not find a single part in database');

        return $part;
    }
}

```

ユーザー入力検証用のヘルパークラス

Helpers/ValidationHelper.php

```php
<?php

namespace Helpers;

class ValidationHelper
{
    public static function integer($value, float $min = -INF, float $max = INF): int
    {
        // 値が整数かどうかを検証
        // filter_var()について ...  https://www.php.net/manual/en/filter.filters.validate.php
        $value = filter_var($value, FILTER_VALIDATE_INT, ["min_range" => (int) $min, "max_range" => (int) $max]);

        if ($value === false)
            throw new \InvalidArgumentException("The provided value is not a valid integer.");

        return $value;
    }
}

```

*ViewファイルをView/componentディレクトリに格納

- サーバを立てて、[http://127.0.0.1:8000/parts?id=](http://127.0.0.1:8000/parts?id=1)34にアクセス
- ３４番目のパーツが取得・表示された

DB（M）、Renderer（V）、Routes（C）に処理の役割を分割できた

- ブラウザ側でレンダリング（CSR）するHTML

client.html

```php
<!doctype html>
<html lang="en">

<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet"
        crossorigin="anonymous">

    <title>My Computer Parts Store</title>
</head>

<body>

    <main class="container mt-5 mb-5">
        <!-- 内容はここに挿入されます。 -->
        <div id="content"></div>
    </main>

    <footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2);">
            © 2023:
            <a class="text-dark" href="/">MyComputerPartsStore.com</a>
        </div>
    </footer>

    <script>
        document.addEventListener('DOMContentLoaded', function () {
            // ロード時に、id1のパーツをGETリクエストで取得
            fetch('http://127.0.0.1:8000/api/parts?id=1')
                .then(response => response.json())
                .then(data => {
                    const part = data.part;
                    const html = `
                    <div class="card" style="width: 18rem;">
                        <div class="card-body">
                            <h5 class="card-title">${part.name}</h5>
                            <h6 class="card-subtitle mb-2 text-muted">${part.type} - ${part.brand}</h6>
                            <p class="card-text">
                                <strong>Model:</strong> ${part.model_number}<br />
                                <strong>Release Date:</strong> ${part.release_date}<br />
                                <strong>Description:</strong> ${part.description}<br />
                                <strong>Performance Score:</strong> ${part.performance_score}<br />
                                <strong>Market Price:</strong> $${part.market_price}<br />
                                <strong>RSM:</strong> $${part.rsm}<br />
                                <strong>Power Consumption:</strong> ${part.power_consumptionw}W<br />
                                <strong>Dimensions:</strong> ${part.lengthm}m x ${part.widthm}m x ${part.heightm}m<br />
                                <strong>Lifespan:</strong> ${part.lifespan} years<br />
                            </p>
                            <p class="card-text"><small class="text-muted">Last updated on ${part.updated_at}</small></p>
                        </div>
                    </div>`;

                    // カードHTMLをページに挿入
                    document.getElementById('content').innerHTML = html;
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error);
                    document.getElementById('content').innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>';
                });
        });
    </script>
</body>

</html>
```

- サーバのrootパスであるindex.php(localhost:8000)からclient.html(別のドメイン)にレスポンスを返す場合、異なるオリジン間の通信になる
- ⇒ index.phpに異なるオリジン間の通信を許可する記述をする

```php
// あらゆるドメインからの通信を許可
// * 本番環境では消して、アクセス可能なドメインは制限する
header('Access-Control-Allow-Origin: *');

```

random/part, parts用のボタン追加

client.html

```php
<!doctype html>
<html lang="en">

<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet"
        crossorigin="anonymous">

    <title>My Computer Parts Store</title>
</head>

<body>

    <main class="container mt-5 mb-5">
        <div class="d-flex justify-content-center my-3 buttons">
            <div class="d-flex random-part px-3">
                <button id="randomBtn" type="button" class="btn btn-secondary random-part-button">Get Random
                    Part</button>
            </div>
            <div class="d-flex id-part">
                <input type="text" id="partId" class="form-control" placeholder="Enter Part ID">
                <button id="idBtn" type="button" class="btn btn-secondary id-part-button">Get Part from ID</button>
            </div>
        </div>
        <!-- 内容はここに挿入されます。 -->
        <div id="content"></div>
    </main>

    <footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
            © 2023:
            <a class="text-dark" href="/">MyComputerPartsStore.com</a>
        </div>
    </footer>

    <script>
        const partCard = (part) => {
            return `
                    <div class="card" style="width: 18rem">
                        <div class="card-body">
                            <h5 class="card-title">${part.name}</h5>
                            <h6 class="card-subtitle mb-2 text-muted">${part.type} - ${part.brand}</h6>
                            <p class="card-text">
                                <strong>Model:</strong> ${part.model_number}<br />
                                <strong>Release Date:</strong> ${part.release_date}<br />
                                <strong>Description:</strong> ${part.description}<br />
                                <strong>Performance Score:</strong> ${part.performance_score}<br />
                                <strong>Market Price:</strong> $${part.market_price}<br />
                                <strong>RSM:</strong> $${part.rsm}<br />
                                <strong>Power Consumption:</strong> ${part.power_consumptionw}W<br />
                                <strong>Dimensions:</strong> ${part.lengthm}m x ${part.widthm}m x ${part.heightm}m<br />
                                <strong>Lifespan:</strong> ${part.lifespan} years<br />
                            </p>
                            <p class="card-text"><small class="text-muted">Last updated on ${part.updated_at}</small></p>
                        </div>
                    </div>
            `
        }

        document.addEventListener('DOMContentLoaded', function () {
            // ロード時に、id1のパーツをGETリクエストで取得
            fetch('http://127.0.0.1:8000/api/parts?id=1')
                .then(response => {
                    console.log(response)
                    if (!response.ok) {
                        throw new Error('Network response was not ok')
                    }
                    return response.json()

                })
                .then(data => {
                    const part = data.part

                    // カードHTMLをページに挿入
                    document.getElementById('content').innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    document.getElementById('content').innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // ランダムパーツボタンのクリックイベント
        const randomPartButton = document.getElementById('randomBtn')
        randomPartButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            content.innerHTML = ''
            fetch('http://127.0.0.1:8000/api/random/part')
                .then(res => res.json())
                .then(data => {
                    const part = data.part
                    content.innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // IDでパーツを取得するボタンのクリックイベント
        const idPartButton = document.getElementById('idBtn')
        idPartButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const partId = document.getElementById('partId').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/parts?id=${partId}`)
                .then(res => res.json())
                .then(data => {
                    const part = data.part
                    content.innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

    </script>
</body>

</html>
```

URLバーに(apiじゃないほうの)パスを入力してパーツを表示する場合はHTTPRendererによって表示され、 

client.htmlのボタンからパーツにアクセスする場合はJSONRendererを使ってJSONデータが取得され、client.htmlから表示する

---

エンドポイントを追加

GET /types … 指定したカテゴリのコンピュータパーツのリストを取得

- /types?type={type}&page={paginationPageCount}&perpage={countPerPage}
- type: 取得するコンピュータパーツのカテゴリ
- page: 取得する結果のページ番号
- perpage: 1ページ当たりに表示するアイテム数

ページネーションのオフセットを算出し、LIMIT {offset}, {count}でデータを指定する

Router/routes.php

```php
    ...,
    'api/types' => function () {
        $type = $_GET['type'] ?? null;
        $page = $_GET['page'] ?? 1;
        $perpage = $_GET['perpage'] ?? 3;

        $parts = DatabaseHelper::getComputerPartsByType($type, $page, $perpage);
        return new JSONRenderer(['parts' => $parts]);
    },

```

Helpers/DatabaseHelper.php

```php
    public static function getComputerPartsByType(string $type, int $page, int $perpage): array
    {
        $db = new MySQLWrapper();

        // ページネーションのためのオフセットを計算
        $offset = ($page - 1) * $perpage;
        
        $stmt = $db->prepare("SELECT * FROM computer_parts WHERE type = ? LIMIT ?, ?");
        $stmt->bind_param('sii', $type, $offset, $perpage);
        $stmt->execute();

        $result = $stmt->get_result();
        $parts = $result->fetch_all(MYSQLI_ASSOC);

        if (!$parts)
            throw new Exception('Could not find any parts in database');

        return $parts;
    }

```

client.html

```php
        // 特定カテゴリのパーツを取得するボタンのクリックイベント
        const typePartsButton = document.getElementById('typeBtn')
        typePartsButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const partType = document.getElementById('partType').value
            const page = document.getElementById('page').value
            const perpage = document.getElementById('perpage').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/types?type=${partType}&page=${page}&perpage=${perpage}`)
                .then(res => res.json())
                .then(data => {
                    const parts = data.parts
                    parts.forEach(part => {
                        content.innerHTML += partCard(part)
                    })
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

```

GET /random/computer … 複数のパーツからランダムにコンピュータを生成して取得

- /random/computer

GET /parts/newest … 最近追加されたパーツを取得

- page
- perpage
- 例：/parts/newest?page=2&perpage=10

GET /parts/performance … パフォーマンス順で上位/下位50位までのパーツを取得

- order … 昇順/降順
- type
- 例: /parts/performance?order=desc&type=CPU

Router/routes.php

```php
<?php

use Helpers\DatabaseHelper;
use Helpers\ValidationHelper;
use Response\HTTPRenderer;
use Response\Render\HTMLRenderer;
use Response\Render\JSONRenderer;

return [
    'random/part' => function (): HTTPRenderer {
        $part = DatabaseHelper::getRandomComputerPart();

        return new HTMLRenderer('component/random-part', ['part' => $part]);
    },
    'parts' => function (): HTTPRenderer {
        // IDの検証
        $id = ValidationHelper::integer($_GET['id'] ?? null);

        $part = DatabaseHelper::getComputerPartById($id);
        return new HTMLRenderer('component/parts', ['part' => $part]);
    },
    'types' => function (): HTTPRenderer {
        $type = $_GET['type'] ?? null;
        $page = $_GET['page'] ?? 1;
        $perpage = $_GET['perpage'] ?? 3;

        $parts = DatabaseHelper::getComputerPartsByType($type, $page, $perpage);
        return new HTMLRenderer('component/parts-list', ['parts' => $parts]);
    },
    'random/computer' => function (): HTTPRenderer {
        $parts = DatabaseHelper::getRandomComputer();
        return new HTMLRenderer('component/computer', ['parts' => $parts]);
    },
    'parts/newest' => function (): HTTPRenderer {
        $page = $_GET['page'] ?? 1;
        $perpage = $_GET['perpage'] ?? 3;

        $parts = DatabaseHelper::getNewestComputerParts($page, $perpage);
        return new HTMLRenderer('component/parts-list', ['parts' => $parts]);
    },
    'parts/performance' => function (): HTTPRenderer {
        $order = $_GET['order'] ?? 'asc';
        $type = $_GET['type'] ?? null;

        $parts = DatabaseHelper::getComputerPartsByPerformance($order, $type, $page, $perpage);
        return new HTMLRenderer('component/parts-list', ['parts' => $parts]);
    },
    'api/random/part' => function (): HTTPRenderer {
        $part = DatabaseHelper::getRandomComputerPart();
        return new JSONRenderer(['part' => $part]);
    },
    'api/parts' => function () {
        $id = ValidationHelper::integer($_GET['id'] ?? null);
        $part = DatabaseHelper::getComputerPartById($id);
        return new JSONRenderer(['part' => $part]);
    },
    'api/types' => function () {
        $type = $_GET['type'] ?? null;
        $page = $_GET['page'] ?? 1;
        $perpage = $_GET['perpage'] ?? 3;

        $parts = DatabaseHelper::getComputerPartsByType($type, $page, $perpage);
        return new JSONRenderer(['parts' => $parts]);
    },
    'api/random/computer' => function (){
        $parts = DatabaseHelper::getRandomComputer();
        return new JSONRenderer(['parts' => $parts]);
    },
    'api/parts/newest' => function () {
        $page = $_GET['page'] ?? 1;
        $perpage = $_GET['perpage'] ?? 3;

        $parts = DatabaseHelper::getNewestComputerParts($page, $perpage);
        return new JSONRenderer(['parts' => $parts]);
    },
    'api/parts/performance' => function () {
        $order = $_GET['order'] ?? 'asc';
        $type = $_GET['type'] ?? null;

        $parts = DatabaseHelper::getComputerPartsByPerformance($order, $type);
        return new JSONRenderer(['parts' => $parts]);
    },
];

```

Helpers/DatabaseHelper.php

```php
<?php

namespace Helpers;

use Database\MySQLWrapper;
use Exception;

class DatabaseHelper
{
    public static function getRandomComputerPart(): array
    {
        $db = new MySQLWrapper();

        $stmt = $db->prepare("SELECT * FROM computer_parts ORDER BY RAND() LIMIT 1");
        $stmt->execute();
        $result = $stmt->get_result();
        $part = $result->fetch_assoc();

        if (!$part)
            throw new Exception('Could not find a single part in database');

        return $part;
    }

    public static function getComputerPartById(int $id): array
    {
        $db = new MySQLWrapper();

        $stmt = $db->prepare("SELECT * FROM computer_parts WHERE id = ?");
        $stmt->bind_param('i', $id);
        $stmt->execute();

        $result = $stmt->get_result();
        $part = $result->fetch_assoc();

        if (!$part)
            throw new Exception('Could not find a single part in database');

        return $part;
    }

    public static function getComputerPartsByType(string $type, int $page, int $perpage): array
    {
        $db = new MySQLWrapper();

        // ページネーションのためのオフセットを計算
        $offset = ($page - 1) * $perpage;
        
        $stmt = $db->prepare("SELECT * FROM computer_parts WHERE type = ? LIMIT ?, ?");
        $stmt->bind_param('sii', $type, $offset, $perpage);
        $stmt->execute();

        $result = $stmt->get_result();
        $parts = $result->fetch_all(MYSQLI_ASSOC);

        if (!$parts)
            throw new Exception('Could not find any parts in database');

        return $parts;
    }

    public static function getRandomComputer(): array {
        $db = new MySQLWrapper();

        $parts = [
            'SELECT * FROM computer_parts WHERE type = "CPU" ORDER BY RAND() LIMIT 1',
            'SELECT * FROM computer_parts WHERE type = "GPU" ORDER BY RAND() LIMIT 1',              
            'SELECT * FROM computer_parts WHERE type = "SSD" ORDER BY RAND() LIMIT 1',
            'SELECT * FROM computer_parts WHERE type = "RAM" ORDER BY RAND() LIMIT 1',
            'SELECT * FROM computer_parts WHERE type = "Motherboard" ORDER BY RAND() LIMIT 1',
            'SELECT * FROM computer_parts WHERE type = "PSU" ORDER BY RAND() LIMIT 1',
            'SELECT * FROM computer_parts WHERE type = "Case" ORDER BY RAND() LIMIT 1'
        ];

        foreach ($parts as $key => $query) {
            $stmt = $db->prepare($query);
            $stmt->execute();
            $result = $stmt->get_result();
            $part = $result->fetch_assoc();
            $parts[$key] = $part;
        }

        return $parts;
    }

    public static function getNewestComputerParts(int $page, int $perpage): array
    {
        $db = new MySQLWrapper();

        $offset = ($page - 1) * $perpage;

        // created_atで降順に並べ替え、ページネーションを適用
        $stmt = $db->prepare("SELECT * FROM computer_parts ORDER BY created_at DESC LIMIT ?, ?");
        $stmt->bind_param('ii', $offset, $perpage);
        $stmt->execute();

        $result = $stmt->get_result();
        $parts = $result->fetch_all(MYSQLI_ASSOC);

        if (!$parts)
            throw new Exception('Could not find any parts in database');

        return $parts;
    }

    public static function getComputerPartsByPerformance(string $order, string $type) {
        $db = new MySQLWrapper();

        $stmt = $order == 'asc' ?
            $db->prepare("SELECT * FROM computer_parts WHERE type = ? ORDER BY performance_score ASC LIMIT 50") :
            $db->prepare("SELECT * FROM computer_parts WHERE type = ? ORDER BY performance_score DESC LIMIT 50");
        $stmt->bind_param('s', $type);
        $stmt->execute();

        $result = $stmt->get_result();
        $parts = $result->fetch_all(MYSQLI_ASSOC);

        if (!$parts)
            throw new Exception('Could not find any parts in database');

        return $parts;
    }
}

```

client.html

```php
<!doctype html>
<html lang="en">

<head>
    <!-- Required meta tags -->
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <!-- Bootstrap CSS -->
    <link href="https://cdn.jsdelivr.net/npm/bootstrap@5.0.2/dist/css/bootstrap.min.css" rel="stylesheet"
        crossorigin="anonymous">

    <title>My Computer Parts Store</title>
</head>

<body>

    <main class="container mt-5 mb-5">
        <div class="d-flex flex-column justify-content-center my-3 buttons">
            <div class="d-flex random-part px-3">
                <button id="randomBtn" type="button" class="btn btn-secondary random-part-button">Get Random
                    Part</button>
            </div>
            <div class="d-flex id-part">
                <input type="text" id="partId" class="form-control" placeholder="Enter Part ID">
                <button id="idBtn" type="button" class="btn btn-secondary id-part-button">Get Part from ID</button>
            </div>
            <div class="d-flex type-parts">
                <input type="text" id="partType" class="form-control" placeholder="Enter Part Type">
                <input type="text" id="page" class="form-control" placeholder="Enter Page Number">
                <input type="text" id="perpage" class="form-control" placeholder="Enter Item Counts per Page">
                <button id="typeBtn" type="button" class="btn btn-secondary type-parts-button">Get Parts of a particular
                    Type</button>
            </div>
            <div class="d-flex random-computer">
                <button id="randomComputerBtn" type="button" class="btn btn-secondary random-computer-button">Get
                    Today's Random Computer</button>
            </div>
            <div class="d-flex newest-parts">
                <input type="text" id="page" class="form-control" placeholder="Enter Page Number">
                <input type="text" id="perpage" class="form-control" placeholder="Enter Item Counts per Page">
                <button id="newestBtn" type="button" class="btn btn-secondary newest-parts-button">Get Newest
                    Parts</button>
            </div>
            <div class="d-flex performance-parts">
                <div class="d-flex flex-column">
                    <input type="radio" id="asc" name="performance" value="asc">
                    <label for="asc">Ascending</label>
                    <input type="radio" id="desc" name="performance" value="desc">
                    <label for="desc">Descending</label>
                </div>
                <input type="text" id="type" class="form-control" placeholder="Enter Part Type">
                <button id="performanceBtn" type="button" class="btn btn-secondary performance-parts-button">Get Parts
                    by Performance</button>
            </div>
        </div>
        <!-- 内容はここに挿入されます。 -->
        <div id="content"></div>
    </main>

    <footer class="bg-light text-center text-lg-start">
        <div class="text-center p-3" style="background-color: rgba(0, 0, 0, 0.2)">
            © 2023:
            <a class="text-dark" href="/">MyComputerPartsStore.com</a>
        </div>
    </footer>

    <script>
        const partCard = (part) => {
            return `
                    <div class="card" style="width: 18rem">
                        <div class="card-body">
                            <h5 class="card-title">${part.name}</h5>
                            <h6 class="card-subtitle mb-2 text-muted">${part.type} - ${part.brand}</h6>
                            <p class="card-text">
                                <strong>Model:</strong> ${part.model_number}<br />
                                <strong>Release Date:</strong> ${part.release_date}<br />
                                <strong>Description:</strong> ${part.description}<br />
                                <strong>Performance Score:</strong> ${part.performance_score}<br />
                                <strong>Market Price:</strong> $${part.market_price}<br />
                                <strong>RSM:</strong> $${part.rsm}<br />
                                <strong>Power Consumption:</strong> ${part.power_consumptionw}W<br />
                                <strong>Dimensions:</strong> ${part.lengthm}m x ${part.widthm}m x ${part.heightm}m<br />
                                <strong>Lifespan:</strong> ${part.lifespan} years<br />
                            </p>
                            <p class="card-text"><small class="text-muted">Last updated on ${part.updated_at}</small></p>
                        </div>
                    </div>
            `
        }

        document.addEventListener('DOMContentLoaded', function () {
            // ロード時に、id1のパーツをGETリクエストで取得
            fetch('http://127.0.0.1:8000/api/parts?id=1')
                .then(response => {
                    console.log(response)
                    if (!response.ok) {
                        throw new Error('Network response was not ok')
                    }
                    return response.json()

                })
                .then(data => {
                    const part = data.part

                    // カードHTMLをページに挿入
                    document.getElementById('content').innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    document.getElementById('content').innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // ランダムパーツボタンのクリックイベント
        const randomPartButton = document.getElementById('randomBtn')
        randomPartButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            content.innerHTML = ''
            fetch('http://127.0.0.1:8000/api/random/part')
                .then(res => res.json())
                .then(data => {
                    const part = data.part
                    content.innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // IDでパーツを取得するボタンのクリックイベント
        const idPartButton = document.getElementById('idBtn')
        idPartButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const partId = document.getElementById('partId').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/parts?id=${partId}`)
                .then(res => res.json())
                .then(data => {
                    const part = data.part
                    content.innerHTML = partCard(part)
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // 特定カテゴリのパーツを取得するボタンのクリックイベント
        const typePartsButton = document.getElementById('typeBtn')
        typePartsButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const partType = document.getElementById('partType').value
            const page = document.getElementById('page').value
            const perpage = document.getElementById('perpage').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/types?type=${partType}&page=${page}&perpage=${perpage}`)
                .then(res => res.json())
                .then(data => {
                    const parts = data.parts
                    parts.forEach(part => {
                        content.innerHTML += partCard(part)
                    })
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // ランダムでコンピュータを取得するボタンのクリックイベント
        const randomComputerButton = document.getElementById('randomComputerBtn')
        randomComputerButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            content.innerHTML = ''
            fetch('http://127.0.0.1:8000/api/random/computer')
                .then(res => res.json())
                .then(data => {
                    const parts = data.parts
                    parts.forEach(part => {
                        content.innerHTML += partCard(part)
                    })
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // 最新のパーツを取得するボタンのクリックイベント
        const newestPartsButton = document.getElementById('newestBtn')
        newestPartsButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const page = document.getElementById('page').value
            const perpage = document.getElementById('perpage').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/parts/newest?page=${page}&perpage=${perpage}`)
                .then(res => res.json())
                .then(data => {
                    const parts = data.parts
                    parts.forEach(part => {
                        content.innerHTML += partCard(part)
                    })
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })

        // パフォーマンス数値順でパーツを取得するボタンのクリックイベント
        const performancePartsButton = document.getElementById('performanceBtn')
        performancePartsButton.addEventListener('click', () => {
            const content = document.getElementById('content')
            const type = document.getElementById('type').value
            const order = document.querySelector('input[name="performance"]:checked').value
            content.innerHTML = ''
            fetch(`http://127.0.0.1:8000/api/parts/performance?order=${order}&type=${type}`)
                .then(res => res.json())
                .then(data => {
                    const parts = data.parts
                    parts.forEach(part => {
                        content.innerHTML += partCard(part)
                    })
                })
                .catch(error => {
                    console.error('There was an error fetching the data:', error)
                    content.innerHTML = '<div class="alert alert-danger">An error occurred while fetching data.</div>'
                })
        })
    </script>
</body>

</html>
```

---

Text Snippet Share
