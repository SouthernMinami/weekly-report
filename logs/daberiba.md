# 作業ログ：ダベリバ(匿名掲示板アプリ)

- 画像と文章を投稿できる
- メインスレッドに対しても画像と文章で返信できる

### postsテーブル

- [ ]  id: int
- [ ]  parent_id: int(nullable)
- [ ]  title: varchar(255)(nullable)
- [ ]  content: text
- [ ]  path: varchar(255)
- [ ]  image_path: varchar(255)
- [ ]  thumbnail_path: varchar(255)
- [ ]  created_at: datetime
- [ ]  updated_at: datetime

Post has Posts(複数のリプライ)

*メインスレッドの場合はreply_to_idはnull

*リプライの場合、reply_to_idは返信先の投稿のid

### postモデル作成

Models/Post.php

```php
<?php

namespace Models;

use Models\Interfaces\Model;
use Models\Traits\GenericModel;

class Post implements Model {
    use GenericModel;

    public function __construct(
        private ?int $id = null,
        private ?int $parentId = null,
        private ?string $title = null,
        private ?string $content = null,
        private ?string $path = null,
        private ?string $image_path = null,
        private ?string $thumbnail_path = null,
        private ?DataTimeStamp $timeStamp = null
    ){}

    public function getId(): ?int
    {
        return $this->id;
    }

    public function setId(?int $id): void
    {
        $this->id = $id;
    }

    public function getParentId(): ?int
    {
        return $this->parentId;
    }

    public function setParentId(?int $parentId): void
    {
        $this->parentId = $parentId;
    }

    public function getTitle(): ?string
    {
        return $this->title;
    }

    public function setTitle(?string $title): void
    {
        $this->title = $title;
    }

    public function getContent(): ?string
    {
        return $this->content;
    }

    public function setContent(?string $content): void
    {
        $this->content = $content;
    }

    public function getPath(): ?string
    {
        return $this->path;
    }

    public function setPath(?string $path): void
    {
        $this->path = $path;
    }

    public function getImagePath(): ?string
    {
        return $this->image_path;
    }

    public function setImagePath(?string $image_path): void
    {
        $this->image_path = $image_path;
    }

    public function getThumbnailPath(): ?string
    {
        return $this->thumbnail_path;
    }

    public function setThumbnailPath(?string $thumbnail_path): void
    {
        $this->thumbnail_path = $thumbnail_path;
    }

    public function getTimeStamp(): ?DataTimeStamp
    {
        return $this->timeStamp;
    }

    public function setTimeStamp(?DataTimeStamp $timeStamp): void
    {
        $this->timeStamp = $timeStamp;
    }
}

```

Database/DataAccess/Interfaces/PostDAO.php

```php
<?php

namespace Database\DataAccess\Interfaces;

use Models\Post;

interface PostDAO {
    public function create(Post $postData): bool;
    public function getById(int $id): ?Post;
    public function update(Post $postData): bool;
    public function delete(int $id): bool;
    public function createOrUpdate(Post $postData): bool;

    /**
     * @param int $offset
     * @param int $limit
     * @return Post[] ほか投稿への返信ではないメインスレッドであるすべての投稿、つまりparentIdがnullの投稿
     */
    public function getAllThreads(int $offset, int $limit): array;
    
    /**
     * @param Post $postData - すべての返信の親となるメインスレッド（投稿）
     * @param int $offset
     * @param int $limit
     * @return Post[] $parentId が $postData の id と一致する投稿の返信
     */
    public function getReplies(Post $postData, int $offset, int $limit): array;
}

```

Database/DataAccess/Implementations/PostDAOImpl.php

```php
<?php

namespace Database\DataAccess\Implementations;

use Database\DataAccess\Interfaces\PostDAO;
use Database\DatabaseManager;
use Models\Post;
use Models\DataTimeStamp;

class PostDAOImpl implements PostDAO
{
    public function create(Post $postData): bool
    {
        if($postData->getId() !== null) throw new \Exception('Failed to create post: Cannot create post with existing ID ' . $postData->getId());
        return $this->createOrUpdate($postData);
    }

    public function getById(int $id): ?Post
    {
        $mysqli = DatabaseManager::getMysqliConnection();
        $post = $mysqli->prepareAndFetchAll("SELECT * FROM posts WHERE id = ?",'i',[$id])[0]??null;

        return $post === null ? null : $this->resultToPost($post);
    }

    public function update(Post $postData): bool
    {
        if($postData->getId() === null) throw new \Exception('Computer part specified has no ID.');

        $current = $this->getById($postData->getId());
        if($current === null) throw new \Exception(sprintf("Computer part %s does not exist.", $postData->getId()));

        return $this->createOrUpdate($postData);
    }

    public function delete(int $id): bool
    {
        $mysqli = DatabaseManager::getMysqliConnection();
        return $mysqli->prepareAndExecute("DELETE FROM posts WHERE id = ?", 'i', [$id]);
    }

    public function createOrUpdate(Post $postData): bool
    {
        $mysqli = DatabaseManager::getMysqliConnection();

        $query =
        <<<SQL
            INSERT INTO posts (id, parent_id, title, content, path, thumbnail_path, image_path)
            VALUES (?, ?, ?, ?, ?, ?)
            ON DUPLICATE KEY UPDATE id = ?,
            parent_id = VALUES(parent_id),
            title = VALUES(title),
            content = VALUES(content),
            path = VALUES(path),
            thumbnail_path = VALUES(thumbnail_path),
            image_path = VALUES(image_path)
        SQL;

        $result = $mysqli->prepareAndExecute(
            $query,
            'iisssssi',
            [
                // idがnullの場合、createのほうを行うことになる
                // それ以外の場合、ON DUPLICATE KEY UPDATEによってすでに存在するレコードに対してupdateが行われる
                $postData->getId(), // null idに対しては、mysqlによって自動生成されインクリメントされる
                $postData->getParentId(),
                $postData->getTitle(),
                $postData->getContent(),
                $postData->getPath(),
                $postData->getImagePath(),
                $postData->getThumbnailPath(),
                // ON DUPLICATE KEY UPDATEのために再度idを指定
                $postData->getId()
            ],
        );

        if(!$result) return false;

        // createの場合、idがnullなのでmysqlによって自動生成されたidをセットする
        if($postData->getId() === null){
            $postData->setId($mysqli->insert_id);
            $timeStamp = $postData->getTimeStamp()??new DataTimeStamp(date('Y-m-h'), date('Y-m-h'));
            $postData->setTimeStamp($timeStamp);
        }

        return true;
    }

    private function resultToPost(array $data): Post{
        return new Post(
            $data['id'],
            $data['parent_id'],
            $data['title'],
            $data['content'],
            $data['path'],
            $data['image_path'],
            $data['thumbnail_path'],
            new DataTimeStamp($data['created_at'], $data['updated_at'])
        );
    }

    private function resultToPosts(array $results): array{
        $posts = [];

        foreach($results as $result){
            $posts[] = $this->resultToPost($result);
        }

        return $posts;
    }

    public function getAllThreads(int $offset, int $limit): array
    {
        $mysqli = DatabaseManager::getMysqliConnection();

        $query = "SELECT * FROM posts LIMIT ?, ? WHERE parent_id IS NULL";

        $results = $mysqli->prepareAndFetchAll($query, 'ii', [$offset, $limit]);

        return $results === null ? [] : $this->resultToPosts($results);
    }
    
    public function getReplies(Post $postData, int $offset, int $limit): array
    {
        $mysqli = DatabaseManager::getMysqliConnection();

        $query = "SELECT * FROM posts WHERE parent_id = ? LIMIT ?, ?";

        $results = $mysqli->prepareAndFetchAll($query, 'iii', [$postData->getId(), $offset, $limit]);

        return $results === null ? [] : $this->resultToPosts($results);
    }
 
}

```

[https://qiita.com/Yuki_Oshima/items/2a73cf70ccbf67bd5215](https://qiita.com/Yuki_Oshima/items/2a73cf70ccbf67bd5215)

- php console code-gen migration --name CreatePostsTable

Database/Migrations/{timestamp}_CreatePostsTable.php

```php
<?php

namespace Database\Migrations;

use Database\SchemaMigration;

class CreatePostsTable implements SchemaMigration
{
    public function up(): array
    {
        // マイグレーション処理を書く
        return [
            'CREATE TABLE posts (
                id INT AUTO_INCREMENT PRIMARY KEY,
                parent_id INT DEFAULT NULL,
                title VARCHAR(255) DEFAULT NULL,
                content TEXT,
                path VARCHAR(255),
                image_path VARCHAR(255),
                thumbnail_path VARCHAR(255),
                created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP
            )'
        ];
    }

    public function down(): array
    {
        // ロールバック処理を書く
        return [
            'DROP TABLE posts'
        ];
    }
}
```

- `php console migrate --init`
- `composer require fakerphp/faker`

fakerでダミーデータ挿入

Database/Seeds/PostsSeeder.php

```php
<?php

namespace Database\Seeds;

require_once 'vendor/autoload.php';

use Database\AbstractSeeder;

class PostsSeeder extends AbstractSeeder
{
    protected ?string $tableName = 'posts';
    protected array $tableColumns = [
        [
            'data_type' => 'int',
            'column_name' => 'parent_id'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'title'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'content'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'image_path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'thumbnail_path'
        ]
    ];

    public function createRowData(): array
    {
        return [
            ...array_map(function () {
                return [
                    0,
                    \Faker\Factory::create()->title,
                    \Faker\Factory::create()->text,
                    hash('sha256', \Faker\Factory::create()->text),
                    \Faker\Factory::create()->imageUrl,
                    \Faker\Factory::create()->imageUrl('200', '200'),
                ];

            }, range(0, 9))
        ];
    }
}

```

- php console seed

仮投稿データ（メインスレッドのみ）の表示
