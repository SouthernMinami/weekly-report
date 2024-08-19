# アプリ開発ログ：のらったー

野良動物画像の共有アプリ

### *EC2インスタンス再起動したら必ずやること

- インスタンスのパブリックIPアドレスが新しいものに変わってるから、namecheapのAdvanced DNSからそのアドレスのA Recordを追加する

---

参考：[imgur](https://imgur.com/)

### imagesテーブル設計

id: string

title: string

image_path: string

post_url: string

delete_url: string

view_count: int

ip_address: string

size: float

created_at: date

accessed_at: date

いつも通り、コマンド作成用クラス、DB操作用クラス、ヘルパークラスを作成 

＆composerをインストール

（省略）

issue作ってから設計等していく

参考記事：

[［GitHub］一人開発でもissueベース/セルフプルリクエストを使って開発する - Qiita](https://qiita.com/braveryk7/items/5208263cd06a8878f0c2)

### テーブルマイグレーション

- DB作成　`mysql -u vboxuser -p` `CREATE DATABASE image_db;`
- `php console code-gen migration --name CreateImagesTable`
- 

### サイトレイアウト

![Untitled](%E3%82%A2%E3%83%95%E3%82%9A%E3%83%AA%E9%96%8B%E7%99%BA%E3%83%AD%E3%82%AF%E3%82%99%EF%BC%9A%E3%81%AE%E3%82%89%E3%81%A3%E3%81%9F%E3%83%BC%20685f1e717be1481f842c80e9114d272e/Untitled.png)

### ルーティングの設定

Routing/routes.php

```php
<?php

use Response\HTTPRenderer;
use Response\Render\HTMLRenderer;
use Response\Render\JSONRenderer;
use Helpers\ValidationHelper;

return [
    'new' => function (): HTMLRenderer {
        return new HTMLRenderer('new');
    },
    // 他のパスに対しても追加していく
    // 必要に応じてJSON形式のデータにアクセスするルートも追加していく
];
```

## 投稿機能

feat/7/post

### 投稿 → imagesテーブルに保存

新規投稿画面

Views/new.php

```php
<?php

namespace Views;

?>

<div class="title-container">
    <h1 class="page-title">NEW POST</h1> 
    <div class="d-flex flex-column p-4 m-3 w-50 border border-secondary rounded upload-container">
        <label for="upload-file" >Upload Image here<br/>(* JPG/JPEG/PNG/GIF, max 4MB)</label>
        <label class="pb-3">
            <span class="btn btn-primary px-5 py-3">
                <i class="fas fa-upload"></i>
                UPLOAD
                <input type="file" id="upload-file" accept=".jpg, .jpeg, .png, .gif" class="pb-3 upload-file" style="display:none"/>        
            </span>
        </label>    
        <div id="upload-error" style="display: none;">            
            <p class="text-danger upload-error"></p>
        </div>
        <label for="post-title">Title</label>
        <input type="text" id="post-title" class="mb-3 w-75" placeholder="eg. My cat (* max 25 characters)" maxlength="25"/>
        <label for="preview">Image Preview</label>
        <div id="preview" class="pb-3 preview">
            <img id="preview-img" class="img-fluid preview-img"/>
        </div>
        <label for="post-description">Description</label>
        <textarea id="post-description" placeholder="eg. My cat is cute. (* max 255 characters)" maxlength="255" class="mb-3 b-3"></textarea>
        <button id="upload-button" class="btn btn-success" onClick="postImage()">POST</button>
    </div>     
</div>
<script src="/public/js/app_new.js"></script>
```

public/js/app_new.js

```jsx
const fileInput = document.getElementById('upload-file')
const uploadErrorMsg = document.getElementById('upload-error')
const previewImage = document.getElementById('preview-img')

window.onload = () => {
    // file inputのアップロード状態とエラーメッセージの表示をリセット
    fileInput.value = ''
    uploadErrorMsg.style.display = 'none' 
}

fileInput.addEventListener('change', (e) => {
    uploadErrorMsg.style.display = 'none'
    previewImage.src = ''

    // キャンセルを押した場合は、ファイルアップロードをリセット
    if (e.target.files.length === 0) {
        return
    }
    
    const reader = new FileReader()
    const file = e.target.files[0]
    // サイズが4MB以下なら、画像ファイルを読み込みbase64形式に変換して、resultに格納
    if (file.size < 4000000) {
        reader.readAsDataURL(file)    
        // file readerのロード時に、previewImageのsrcにreaderのresult属性を代入
        reader.onload = () => {
        previewImage.src = reader.result
    }
    } else {
        // 4MBより大きければエラーメッセージを表示
        uploadErrorMsg.style.display = 'block'
        uploadErrorMsg.textContent = 'Please upload file less than 4MB'
        uploadErrorMsg.classList.add('text-danger')
    }
})

const postImage = () => {    
    const file = fileInput.files[0]
    // 画像が選択されていない場合は、エラーメッセージを表示
    if (!file || file.size > 4000000) {
        console.log('error')
        uploadErrorMsg.style.display = 'block'
        uploadErrorMsg.textContent = 
            !file ? 'Please select an image above' : 'Please upload file less than 4MB'
        uploadErrorMsg.classList.add('text-danger')
        return
    }

    const title = document.getElementById('post-title').value
    const description = document.getElementById('post-description').value

    const formData = new FormData()
    formData.append('title', title)
    formData.append('description', description)
    formData.append('file', file)

    fetch('/Helpers/postImage.php', {
        method: 'POST',
        body: formData
    })
    .then(response => response.text())
    .then(data => {
        console.log(data)
    })
    .catch(error => console.error('Error:', error))
}
```

POSTリクエストをpostImage.phpに送り、ファイルからでSeedコマンドを実行

Helpers/postImage.php

```php
<?php

namespace Helpers;

require_once '../vendor/autoload.php';

use Helpers\ValidationHelper;

$title = ValidationHelper::string(isset($_POST['title']) && $_POST['title'] !== '' ? $_POST['title'] : 'untitled');
$description = ValidationHelper::string(isset($_POST['description']) && $_POST['description'] !== '' ? $_POST['description'] : 'no description');
$date = ValidationHelper::string(date('Y-m-d H:i:s'));
$imageFile = $_FILES['file'];
// imageFileを元にハッシュ値を生成し、一意の閲覧用URLと削除用URLを生成
$postPath = ValidationHelper::string(hash('md5', $imageFile['name'] . $date));
$deletePath = ValidationHelper::string(hash('md5', $imageFile['name'] . $date . 'delete'));
// 投稿したユーザーのデバイスのIPアドレスを取得
$ipAddress = ValidationHelper::string($_SERVER['REMOTE_ADDR']);
$fileSize = $imageFile['size'];

// 画像ファイルを一時ディレクトリに保存
$imagePath = '../Temps/' . $imageFile['name'];
if (move_uploaded_file($imageFile['tmp_name'], $imagePath)) {
    $dataStr = implode(',', [$title, $description, $imagePath, $postPath, $deletePath, $ipAddress, $fileSize]);
    $command = sprintf('php ../console seed --data "%s"', escapeshellarg($dataStr));
    exec($command, $output, $return_var);

    if($return_var !== 0) {
        http_response_code(500);
        echo 'Failed to post due to internal error, please contact the admin. <br>';
        exit();
    }
} else {
    http_response_code(500);
    echo 'Failed to upload the file.';
    exit();
}

// 投稿に成功したら/image/{hash_string}にリダイレクト
header('Location: /image/' . $postPath);
```

⭐️ポイント

- [ ]  あとからimagePath先の画像にアクセスして表示できるように、あらかじめ `move_upload_file($file, $path)` で画像ファイルをディレクトリに保存する
- [ ]  投稿したユーザーのIPアドレスは`$_SERVER['REMOTE_ADDR']`  で取得

Databases/AbstractSeeder.php

```php
<?php

namespace Database\Seeds;

require_once __DIR__ . '/../../vendor/autoload.php';

use Database\AbstractSeeder;
use Database\MySQLWrapper;

class ImagesSeeder extends AbstractSeeder
{

    protected ?string $tableName = 'images';
    protected array $tableColumns = [
        [
            'data_type' => 'string',
            'column_name' => 'title'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'description'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'image_path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'post_path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'delete_path'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'view_count'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'ip_address'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'file_size'
        ],
    ];

    protected MySQLWrapper $db;

    public function createRowData(string $dataStr): array
    {
        $data_array = explode(',', $dataStr);
        
        return [
            [
                $data_array[0],
                $data_array[1],
                $data_array[2],
                $data_array[3],
                $data_array[4],
                0,
                $data_array[5],
                (int)$data_array[6]
            ]
        ];
    }
}
```

Databases/Seeds/ImagesSeeder.php

```php
<?php

namespace Database\Seeds;

require_once __DIR__ . '/../../vendor/autoload.php';

use Database\AbstractSeeder;
use Database\MySQLWrapper;

class ImagesSeeder extends AbstractSeeder
{

    protected ?string $tableName = 'images';
    protected array $tableColumns = [
        [
            'data_type' => 'string',
            'column_name' => 'title'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'description'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'image_path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'post_path'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'delete_path'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'view_count'
        ],
        [
            'data_type' => 'string',
            'column_name' => 'ip_address'
        ],
        [
            'data_type' => 'int',
            'column_name' => 'file_size'
        ],
    ];

    protected MySQLWrapper $db;

    public function createRowData(string $dataStr): array
    {
        $data_array = explode(',', $dataStr);
        
        return [
            [
                $data_array[0],
                $data_array[1],
                $data_array[2],
                $data_array[3],
                $data_array[4],
                0,
                $data_array[5],
                (int)$data_array[6]
            ]
        ];
    }
}
```

### 閲覧ページにポストと削除ボタンを表示

### ３０日間アクセスしてない投稿の自動削除(Cronジョブ）

---

## デプロイ

- ignoreしたファイル、ディレクトリを本番環境のプロジェクトで作成し直す (.env, Temp)
- composer install
- mysqlでimage_dbを作成
    - mysql -u vboxuser -p
    - CREATE DATABASE image_db;
    - USE image_db;
- php console migrate --init
- (テーブル作成できたか確認したい時 USE image_db; SHOW TABLES;)

- サイト公開用フォルダ　`mkdir -p /var/www/noratter/public` → `sudo chown -R $USER:$USER /var/www/noratter/public` (publicの所有者を現在のユーザーに移す)
- プロジェクトフォルダとpublicフォルダをリンク `sudo ln -s ~/web/noratter /var/www/noratter/public`
- シンボリックリンク確認　`ls -l /var/www/noratter/public と ls -l /var/www/noratter/public/noratter`
- 設定ファイル　`sudo nano /etc/nginx/sites-available/noratter.kano.wiki`

```
server {
    listen 80;
    listen [::]:80;

    root /var/www/noratter/public/noratter;
    index index.php;

    server_name noratter.kano.wiki;
		
		location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

		location ~ \.php$ {
	    include snippets/fastcgi-php.conf;
	    fastcgi_pass unix:/run/php/php8.1-fpm.sock;
	}
}
```

- DNS設定をNameCheapで追記
- 設定ファイルへのショートカット　`sudo ln -s /etc/nginx/sites-available/noratter.kano.wiki /etc/nginx/sites-enabled/`
- `sudo reboot`→再接続→`sudo systemctl start nginx`
- http/noratter.kano.wikiにアクセスできるか確認
- certbotでhttps接続設定
    - `sudo certbot --nginx -d noratter.kano.wiki`

certbotコマンド実行後のsite-available

```
    root /var/www/snippeta/public/snippeta;
    index index.php;

    server_name snippeta.kano.wiki;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

```

2024/08/15 11:53:24 [error] 196376#196376: *10 FastCGI sent in stderr: "PHP message: PHP Warning:  move_uploaded_file(home/ubuntu/web/noratter/Temps/1801287.png): Failed to open stream: No such file or directory in /home/ubuntu/web/noratter/Helpers/postImage.php on line 22PHP message: PHP Warning:  move_uploaded_file(): Unable to move "/tmp/phpMr5An8" to "home/ubuntu/web/noratter/Temps/1801287.png" in /home/ubuntu/web/noratter/Helpers/postImage.php on line 22" while reading response header from upstream, client: 133.200.217.128, server: noratter.kano.wiki, request: "POST /Helpers/postImage.php HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.1-fpm.sock:", host: "noratter.kano.wiki", referrer: "[https://noratter.kano.wiki/new](https://noratter.kano.wiki/new)"

2024/08/19 13:43:18 [error] 210253#210253: *7 FastCGI sent in stderr: "PHP message: PHP Warning:  move_uploaded_file(/home/ubuntu/web/noratter/Temps/1801287.png): Failed to open stream: Permission denied in /home/ubuntu/web/noratter/Helpers/postImage.php on line 24PHP message: PHP Warning:  move_uploaded_file(): Unable to move "/tmp/phpz9aCOt" to "/home/ubuntu/web/noratter/Temps/1801287.png" in /home/ubuntu/web/noratter/Helpers/postImage.php on line 24" while reading response header from upstream, client: 133.200.217.128, server: noratter.kano.wiki, request: "POST /Helpers/postImage.php HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.1-fpm.sock:", host: "noratter.kano.wiki", referrer: "[https://noratter.kano.wiki/new](https://noratter.kano.wiki/new)"

2024/08/19 13:43:18 [error] 210253#210253: *7 FastCGI sent in stderr: "PHP message: PHP Fatal error:  Uncaught InvalidArgumentException: No image post found with the path: undefined in /home/ubuntu/web/noratter/Helpers/DatabaseHelper.php:23

Stack trace:

#0 /home/ubuntu/web/noratter/Routing/routes.php(20): Helpers\DatabaseHelper::getImage()

#1 /home/ubuntu/web/noratter/index.php(28): {closure}()

#2 {main}

thrown in /home/ubuntu/web/noratter/Helpers/DatabaseHelper.php on line 23" while reading response header from upstream, client: 133.200.217.128, server: noratter.kano.wiki, request: "GET /image/undefined HTTP/1.1", upstream: "fastcgi://unix:/run/php/php8.1-fpm.sock:", host: "noratter.kano.wiki", referrer: "[https://noratter.kano.wiki/new](https://noratter.kano.wiki/new)"

- 画像の保存ディレクトリ内のファイルにアクセスできるように権限を変更(r w x)

`sudo chmod -R 777 /home/ubuntu/web/noratter/Temps`
