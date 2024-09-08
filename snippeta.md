# アプリ開発ログ： スニペタ

参考：[PASTEBIN](https://pastebin.com/D8EsgzNm)

### snippetsテーブル設計

title : string

language : string

content : string

expiration_date : timestamp

created_at : timestamp

updated_at : timestamp

user_id(FK) : string (会員登録機能作らないなら必要ない）

### mysqliから派生して、DBと通信するためのクラスを作成

Database/MySQLWrapper.php

```php
<?php

namespace Database;

use mysqli;
use Helpers\\Settings;

class MySQLWrapper extends mysqli
{
    public function __construct(?string $hostname = 'localhost', ?string $username = null, ?string $password = null, ?string $database = null, ?int $port = null, ?string $socket = null)
    {
        mysqli_report(MYSQLI_REPORT_ERROR | MYSQLI_REPORT_STRICT);
        $username = $username ?? Settings::env('DATABASE_USER');
        $password = $password ?? Settings::env('DATABASE_USER_PASSWORD');
        $database = $database ?? Settings::env('DATABASE_NAME');

        parent::__construct($hostname, $username, $password, $database, $port, $socket);
    }

    public function getDatabaseName(): string
    {
        return $this->query("SELECT database() AS the_db")->fetch_row()[0];
    }
}

```

DB設定用のヘルパークラス

Helpers/Settings.php

```php
<?php

namespace Helpers;

use Exceptions\\ReadAndParseEnvException;

class Settings
{
    private const ENV_PATH = '.env';

    public static function env(string $pair): string
    {
        $config = parse_ini_file(dirname(__FILE__, 2) . '/' . self::ENV_PATH);

				// envファイル読み込みに失敗した場合の例外
        if ($config === false) {
            throw new ReadAndParseEnvException();
        }

        return $config[$pair];
    }
}

```

.envファイル読み込み・変換失敗時の例外（仮）

Exceptions/ReadAndParseEnvException.php

```php
<?php

namespace Exceptions;

use Exception;

class ReadAndParseEnvException extends Exception
{
}
;

```

### timestamp付きでマイグレーションファイルを作成するためのコマンドを作成　code_gen

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

            // 次のargsエントリがオプション(-)でない場合は引数値として扱う
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

code-genコマンド

Commands/Programs/CodeGeneration.php

```php
<?php

// コード生成のコマンド

namespace Commands\\Programs;

use Commands\\AbstractCommand;
use Commands\\Argument;

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
                $migrationName = $this->getArgValue('name');
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

namespace Database\\Migrations;

use Database\\SchemaMigration;

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
            namespace Commands\\Programs;

            use Commands\\AbstractCommand;
            use Commands\\Argument;

            class $capitalized_name extends AbstractCommand
            {
                protected static ?string \\$alias = '$name';

                public static function getArgs(): array
                {

                    return [
                        " . implode(",\\n", array_map(function ($arg) {
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
        $registry_content = str_replace("return [", "return [\\n    Commands\\Programs\\\\$capitalized_name::class,", $registry_content);
        file_put_contents($registry_path, $registry_content);
    }
}

```

生成するマイグレーションファイルで使うクラスの元になるインタフェース

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

### migrateコマンド、seedコマンドの作成

Commands/Programs/Migrate.php

```php
<?php

// マイグレーションの実行、ロールバック、新しいスキーマインストールを行う
namespace Commands\\Programs;

use Commands\\AbstractCommand;
use Commands\\Argument;
use Database\\MySQLWrapper;

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
            throw new \\Exception("マイグレーションテーブルの作成に失敗しました。");
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
                throw new \\Exception("マイグレーションファイルのクエリが空です。");
            }

            // クエリを実行
            $this->processQueries($queries);
            $this->insertMigration($filename);
        }

        $this->log("マイグレーションが完了しました。\\n");
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
        if (preg_match('/([^_]+)\\.php$/', $filename, $matches)) {
            return sprintf("%s\\%s", 'Database\\Migrations', $matches[1]);
        } else {
            throw new \\Exception("クラス名の取得に失敗しました。");
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

        // デフォルトは降順でファイルをソート
        usort($all_files, function ($a, $b) use ($order) {
            $compare_result = strcmp($a, $b);
            return ($order === 'desc') ? -$compare_result : $compare_result;
        });

        return $all_files;
    }

    private function processQueries(array $queries): void
    {
        $mysqli = new MySQLWrapper();

        foreach ($queries as $query) {
            $result = $mysqli->query($query);
            if (!$result) {
                throw new \\Exception("クエリの実行に失敗しました。");
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
            throw new \\Exception("クエリの準備に失敗しました。");
        }

        // 準備されたクエリに実際のファイル名を挿入
        $statement->bind_param('s', $filename);

        // ステートメントの実行
        if (!$statement->execute()) {
            throw new \\Exception("クエリの実行に失敗しました。");
        }

        // ステートメントを閉じる
        $statement->close();
    }

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
                throw new \\Exception("マイグレーションファイルのクエリが空です。");
            }

            $this->processQueries($queries);
            $this->removeMigration($filename);
            $count++;
        }

        $this->log("ロールバックが完了しました。\\n");
    }

    private function removeMigration(string $filename): void
    {
        $mysqli = new MySQLWrapper();
        $statement = $mysqli->prepare("DELETE FROM migrations WHERE filename = ?");

        if (!$statement) {
            throw new \\Exception("クエリの準備に失敗しました。(" . $mysqli->errno . ")" . $mysqli->error);
        }

        $statement->bind_param('s', $filename);
        if (!$statement->execute()) {
            throw new \\Exception("クエリの実行に失敗しました。(" . $mysqli->errno . ")" . $mysqli->error);
        }

        $statement->close();
    }
}

```

Commands/Programs/Seed.php

```php
<?php

namespace Commands\\Programs;

use Commands\\AbstractCommand;
use Database\\MySQLWrapper;
use Database\\Seeder;

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
                $class_name = 'Database\\Seeds\\\\' . pathinfo($file, PATHINFO_FILENAME);

                // シードファイルを読み込む
                include_once $directory_path . '/' . $file;

                if (class_exists($class_name) && is_subclass_of($class_name, Seeder::class)) {
                    $seeder = new $class_name(new MySQLWrapper());
                    $seeder->seed();
                } else
                    throw new \\Exception('Seeder must be a class that subclasses the seeder interface');
            }
        }
    }
}

```

ダミーデータを挿入して確認するために、fakerを使う

`composer require fakerphp/faker`

composer.json

```php
{
    "require": {
        "fakerphp/faker": "^1.15"
    }
}

```

タイムスタンプの型のためにcarbonを使う

composer require nesbot/carbon

データ挿入ファイル用クラスのもとになるインタフェース

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

require_once 'vendor/autoload.php';

use Database\\MySQLWrapper;
use Carbon\\Carbon;

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
            throw new \\Exception('Class requires a table name');
        if (empty($this->tableColumns))
            throw new \\Exception('Class requires a columns');

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
            throw new \\Exception('Row does not match the ' . $this->tableName . ' table columns.');

        foreach ($row as $i => $value) {
            $columnDataType = $this->tableColumns[$i]['data_type'];
            $columnName = $this->tableColumns[$i]['column_name'];

            if (!isset(static::AVAILABLE_TYPES[$columnDataType]))
                throw new \\InvalidArgumentException(sprintf("The data type %s is not an available data type.", $columnDataType));

            // 値のデータタイプを返すget_debug_type()とgettype()がある
            // <https://www.php.net/manual/en/function.get-debug-type.php>
            // get_debug_type ... ネイティブPHP8のタイプを返す。 (floatsのgettype()の場合は'float'ではなく、'double'を返す)
            if (get_debug_type($value) !== $columnDataType)
                throw new \\InvalidArgumentException(sprintf("Value for %s should be of type %s. Here is the current value: %s", $columnName, $columnDataType, json_encode($value)));
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

Seedコマンド

Commands/Programs/Seed.php

```php

```

使用コマンドを配列として保存するファイル

Commands/registry.php

```php
<?php

// コマンドを登録するためのレジストリ
// consoleはここから読み取る
return [
    Commands\\Programs\\Touch::class,
    Commands\\Programs\\Migrate::class,
    Commands\\Programs\\CodeGeneration::class,
    Commands\\Programs\\DBWipe::class,
    Commands\\Programs\\BookSearch::class,
    Commands\\Programs\\StateMigrate::class,
    Commands\\Programs\\Seed::class,
];

```

issue作ってから設計等していく

参考記事：

[［GitHub］一人開発でもissueベース/セルフプルリクエストを使って開発する - Qiita](https://qiita.com/braveryk7/items/5208263cd06a8878f0c2)
