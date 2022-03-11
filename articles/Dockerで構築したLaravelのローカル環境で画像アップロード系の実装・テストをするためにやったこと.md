[@ucan-labさんの記事](https://qiita.com/ucan-lab/items/56c9dc3cf2e6762672f4)を丸パクリして立ち上げたローカル環境にて、画像アップロード系の実装とテストを入れました。
どんな手順が必要だったか記録します。

なお、本番環境以外では`public`へ画像を保存しようと思っているので、あくまでLaravel内だけで完結する内容となっています。

## PHPコンテナへGDをインストール

PHPで画像をあれこれするのに必要なPHP拡張モジュールがコンテナにインストールされていないので、Dockerfileをこのように編集しました。

```dockerfile
FROM php:8.1-fpm-buster
SHELL ["/bin/bash", "-oeux", "pipefail", "-c"]

ENV COMPOSER_ALLOW_SUPERUSER=1 \
  COMPOSER_HOME=/composer

COPY --from=composer:2.0 /usr/bin/composer /usr/bin/composer

COPY --from=node:12.14 /usr/local/bin /usr/local/bin
COPY --from=node:12.14 /usr/local/lib /usr/local/lib

RUN apt-get update && \
  - apt-get -y install git unzip libzip-dev libicu-dev libonig-dev && \
  + apt-get -y install git unzip libzip-dev libicu-dev libonig-dev libfreetype6-dev libjpeg62-turbo-dev && \
  apt-get clean && \
  rm -rf /var/lib/apt/lists/* && \
  - docker-php-ext-install intl pdo_mysql zip bcmath
  + docker-php-ext-install intl pdo_mysql zip bcmath && \
  + docker-php-ext-configure gd --with-freetype --with-jpeg && \
  + docker-php-ext-install -j$(nproc) gd

COPY ./php.ini /usr/local/etc/php/php.ini

WORKDIR /work
```

末尾へ雑に書き足してしまいましたが、要するに`docker-php-ext-configure`の引数にPHP拡張が依存するパッケージ名を渡せば、あとは`docker-php-ext-install`スクリプトを書き足すことで欲しいPHP拡張をインストールできるよ、ということみたいです。

参考記事（http://psychedelicnekopunch.com/archives/2355）
PHPの公式Docker Imageのページ（https://hub.docker.com/_/php）

## テストコード

今回作りたいのは「ユーザーアイコンのアップロード機能」なので、画像の格納に加え、ファイルパスを`users`テーブルへ保存するところまで行いたいです。

```php
use Illuminate\Http\UploadedFile;
use Illuminate\Support\Facades\Storage;

		/**
     * @test
     */
    public function ユーザーアイコンアップロード()
    {
        $file = UploadedFile::fake()->image('icon.jpg');
        $expect_file_path = 'icon/' . $file->hashName();

        $response = $this->postJson('/api/user/'. $this->user_id . '/icon', [
            'icon_image' => $file
        ]);
        $response
            ->assertStatus(200);
            
        Storage::disk('public')->assertExists($expect_file_path);
        $this->assertDatabaseHas('users', ['icon_path' => $expect_file_path]);
    }
```

ダミーの画像ファイルを生成してPOSTし、`storage/app/public/`の中に渡した画像ファイルが格納されていること、データベースに画像のパスが保存されているかどうかをテストしています。

参考記事（https://readouble.com/laravel/8.x/ja/http-tests.html#testing-file-uploads）

## 実装

POSTした画像は、`$request`インスタンスから`file`メソッドで取り出します。

その際、画像は`UploadedFile`クラスのインスタンスとなって渡されるとのことでしたが、テストコードで作成したフェイクの画像を渡した際には`Illuminate\Http\Testing\File`クラスのインスタンスが渡ってくるようだったので、引数には両方のクラスを受けられるように書きました。

```php
public function uploadUserIconImage(User $user, UploadFile|File $icon_image): Collection
{
  return ($this->app_repository->isEnvironmentalProduction()) ?
    true: // TODO:S3へアップロードする処理を実装する
  	$this->storeImageToStorage($user, $icon_image);
}

private function storeImageToStorage(User $user, UploadFile|File $icon_image): Collection
{
  $icon_path = $this->image_repository->storeImageToStorage($icon_image);
  $this->repository->update($user, input: ['icon_path' => $icon_path]);

  return $this->createUploadedImageCollection($icon_path);
}

private function createUploadedImageCollection(string $icon_path): Collection
{
  return collect([
    'path' => $icon_path
  ]);
}
```

`isEnvironmentalProduction`メソッドは、単に`App`ファサードのメソッドを使って現在の設定が`production`かどうかを真偽値で返しています。

`local`や`staging`など、要するに本番環境以外であれば以降のプライベートメソッドを実行。
ストレージへ画像を格納する処理は、ドキュメント通りこんな感じです。

```php
public function storeImageToStorage(UploadFile|File $image): string
{
  return $image->storeAs('icon', $image->hashName(), 'public');
}
```

こうすると、`storage/app/public/icon`へ画像が保存され、戻り値はそのファイルのパスとなります。
ファイル名を取得するメソッドは`hashName`以外にもありますが、ファイル名という形で不正な値が渡されることも考えうるため、こちらのメソッドを使ってハッシュ化した上で取得することが推奨されています。

S3へのアップロードについては外部APIとやり取りする処理になるので、レスポンスをモックした上でテストを実装する必要があります。なので実装した暁には、別途Postman等で実際の動作を確認する必要がありそうです。