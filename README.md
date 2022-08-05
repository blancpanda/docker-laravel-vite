# Laravel (Vite) 用 Docker 構成

- Laravel 用 Docker 構成 - Jetstream (inertia) を使う前提

## コンテナの起動/停止

### 開発時（Vite 開発サーバ起動）

```sh
docker compose up -d    # 初回起動時 --build
docker compose down
```

### 本番環境用にビルドしたアプリケーションバンドルにアクセスする場合

docker-compose.override.yml を読み込まないように、 docker-compose.yml のみを明示的に指定する

```sh
docker compose -f docker-compose.yml up -d
docker compose down
```

### Compose ファイルの切り替えについて

開発時は docker-compose.override.yml が自動で適用されるので node コンテナで Vite の開発サーバが起動した状態になる  
vue ファイルの変更が即時反映されるようになる

### コマンドのショートカット

コマンドが長いので bin配下にショートカットのためのスクリプトを用意してある

実行できるようにパーミッションを変更する

```sh
chmod u+x bin/*
```

以下のような使い方ができる

```sh
bin/npm install
```

## Laravel のインストール

app コンテナに入り、Laravel と Jetstream をインストール

```sh
docker compose -f docker-compose.yml up -d  # node コンテナを起動しない
docker compose exec app bash
rm .gitkeep
composer create-project --prefer-dist "laravel/laravel=9.*" .
php artisan storage:link
chmod -R 777 storage bootstrap/cache
composer require laravel/jetstream
php artisan jetstream:install inertia
exit
docker compose down # 一旦停止
```

node コンテナで npm install

```sh
bin/npm install
docker compose up -d    # node コンテナ含めて起動 (Vite 開発サーバ)
```

.env を編集してデータベースの接続先を設定

```text:laravel/.env
# :
DB_CONNECTION=mysql
DB_HOST=db
DB_PORT=3306
DB_DATABASE=laravel
DB_USERNAME=phper
DB_PASSWORD=secret
# :
```

app コンテナでマイグレーション

```sh
bin/artisan migrate
```

## vite.config.js について

Vite の開発サーバは node コンテナで起動するが、他のコンテナからアクセスできるように設定する必要がある。  
Laravel インストール後に修正する。

```js:vite.config.js
// ...
export default defineConfig({
  plugins: [
    // ...
  ],
  server: {
    host: '0.0.0.0',    // 別コンテナからのアクセスを可能にする
  },
})

```

参考)  
laravel-vite-plugin の処理で、Vite 開発サーバ起動時に laravel/public/hot ファイルが作成される。  
hot ファイル内のURLをベースとしてアセットのロードが行われるのだが、コンテナ間とブラウザからのアクセスの両方に対応するには http://0.0.0.0:5173 が hot ファイルに出力される必要がある。

## Laravel をこの Docker 構成ごと git 管理下においた場合の clone からの再構築

※ [repository], [project-dir] は各環境に置き換える

```sh
git clone [repository]
cd [project-dir]
chmod u+x bin/*
docker compose up -d --build
docker compose exec app bash
composer install
cp .env.example .env
php artisan key:generate
php artisan storage:link
chmod -R 777 storage bootstrap/cache
php artisan migrate
exit
docker compose down
bin/npm install
docker compose up -d
```

（少し待ってから） http://localhost:8080/ にアクセスして起動を確認

## 本番環境用にビルドする

```sh
bin/npm run build
```

確認は docker-compose.yml のみを指定して起動

```sh
docker compose -f docker-compose.yml up -d
```

http://localhost:8080/ にアクセスする

## VSCode の設定ファイル

- XDebug を使ったデバッグ実行のための構成ファイル .vscode/launch.json を用意してある

## [テーブル定義書](laravel/database/README.md)の生成

db コンテナ内で mysqldump を使って出力したテーブル定義の xml に xslt をあててマークダウンのテーブル定義書を生成できるようにしている

### mysqldumpでテーブル定義のxmlを出力

```sh
docker compose exec db sh
mysqldump --no-data --xml -u root -p laravel > ./work/laravel.xml
Enter password: secret
exit
```

コメントを出力するようにしているので、日本語名や説明を設定しておくとよい。

### xsltをあててマークダウンに変換

```sh
xsltproc -o laravel/database/README.md tables-mdstyle.xsl db-work/laravel.xml
```
