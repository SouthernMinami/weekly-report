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

準備

- CREATE DATABASE table_migrations;
- スクリプト格納用のファイルの形式

{YYYY-MM-DD}{UNIX_TIMESTAMP}{FILENAME}.php

-
