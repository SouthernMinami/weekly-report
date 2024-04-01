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

![Untitled](%E4%BD%9C%E6%A5%AD%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9APJ5%20%E3%82%B5%E3%83%BC%E3%83%8F%E3%82%99%E3%81%A8%E3%83%86%E3%82%99%E3%83%BC%E3%82%BF%E5%B1%A4%20c2d7924ca566460d9e7debf223b036cd/Untitled%206.png)

- tag, categoryの代わりにtaxonomy(分類)とtaxonomy_termテーブルを作成し、postの分類に使う
- subscription(会員の登録状況)をuserから分離してテーブルを追加
- →subscriptionテーブルのup()にusersテーブルのsubscription列削除の処理を追加し、down()で逆に列を追加させるようにする
- taxonomyテーブルのup()でcategory, tag, post_tagテーブルを削除、down()で作成

*SQL文法でやりがちなこと

- CREATE TABLE 文の最後の要素の後にカンマつけて文法エラー

---

状態ベースのスキーマ管理
