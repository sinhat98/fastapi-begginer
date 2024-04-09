# FastAPI入門

## Dockerのインストール
- Mac<br>
  [Docker for Mac](https://hub.docker.com/editions/community/docker-ce-desktop-mac/)をインストール
- Windows<br>
  [Docker for Windows](https://hub.docker.com/editions/community/docker-ce-desktop-windows/)をインストールすることで、同時に docker-compose がインストールされる
- Linux<br>
  curl でダウンロードしたバイナリファイルからインストールが可能

docker-composeのインストールは[こちら](https://docs.docker.jp/compose/install.html)を参照。

```bash
docker-compose --version
```
でバージョン情報が出力されればOK。

## docker image の作成
イメージのbuild

- Dockerfile
- docker-compose.yaml

上のファイルを用意してbuildする。

```bash
docker-compose build
```

## poetryによるパッケージ管理
poetryでPython環境をセットアップする

### poetry init
```bash
docker-compose run \
  --entrypoint "poetry init \
    --name demo-app \
    --dependency fastapi \
    --dependency uvicorn[standard]" \
  demo-app
```
Dockerコンテナ（demo-app）の中で、 `poetry init` コマンドを実行している。
引数として、 `fastapi` と、ASGIサーバーである `uvicorn` をインストールする依存パッケージとして指定しています。


### パッケージのインストール
```bash
docker-compose run --entrypoint "poetry install --no-root" demo-app
```

新しいパッケージを追加したい場合

1. `pyproject.toml`を編集して`poetry lock`を行う。
```sh
docker-compose run --entrypoint "poetry lock" demo-app
docker-compose run --entrypoint "poetry install --no-root" demo-app
```

2. poetry addで追加
```sh
docker-compose run --entrypoint "poetry add <new-package>" demo-app
```


### build
新しいPythonパッケージを追加した場合などは以下のようにイメージを再ビルドするだけで、`pyproject.toml`に含まれている全てのパッケージをインストールすることができる。

```sh
docker-compose build --no-cache
```

## Hello World
```
.
├── api
│   ├── __init__.py
│   └── main.py
├── Dockerfile
├── docker-compose.yaml
├── poetry.lock
└── pyproject.toml
```

```bash
docker-compose up
```
とすると以下のように表示される。

```sh
[+] Running 1/0
 ✔ Container fastapi-begginer-demo-app-1  Created                                                                                                                        0.0s 
Attaching to fastapi-begginer-demo-app-1
fastapi-begginer-demo-app-1  | INFO:     Will watch for changes in these directories: ['/src']
fastapi-begginer-demo-app-1  | INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
fastapi-begginer-demo-app-1  | INFO:     Started reloader process [1] using WatchFiles
fastapi-begginer-demo-app-1  | INFO:     Started server process [10]
fastapi-begginer-demo-app-1  | INFO:     Waiting for application startup.
fastapi-begginer-demo-app-1  | INFO:     Application startup complete.
```
http://localhost:8000/docs にアクセスする。

http://localhost:8000/hello にアクセうすると、`hello world!`と表示される。


## アプリケーションのディレクトリ構造

REST API
REST APIでは、HTTPでやり取りする際に、URLですべての「リソース」を定義します。あるリソースを表すエンドポイントと、HTTPメソッド（GET/POST/PUT/DELETE など）を組み合わせてAPI全体を構成していきます。

HTTPメソッド エンドポイント（{}内はパラメータ）
の形式で書くと、以下のように定義することができます。

`GET /tasks`
`POST /tasks`
`PUT /tasks/{task_id}`
`DELETE /tasks/{task_id}`
`PUT /tasks/{task_id}/done`
`DELETE /tasks/{task_id}/done`

ディレクトリ構造は下記のようにする。
```
api
├── __init__.py
├── main.py
├── schemas
│   └── __init__.py
├── routers
│   └── __init__.py
├── models
│   └── __init__.py
└── cruds
    └── __init__.py
```


```bash
for dir in "schemas routers models cruds"; then
    mkdir -p $dir
    touch "${dir}/__init__.py"
done

```


## Routers

パスオペレーション関数の作成

TODOアプリの場合、 `/tasks` と `/tasks/{task_id}/done` の2つのリソースに大別できるので、それぞれを、 
`api/routers/task.py` と `api/routers/done.py` に書いていくことにする


## Schemas
### 型ヒント
Pythonでは 「型ヒント（Type Hint）」 を使って関数のシグネチャなどに型を付与することが出来る

通常、型ヒントは実行時に影響を及ぼさず（コードの中身には何も作用せず）、IDEなどに型の情報を与えるものである。
しかし、FastAPIでは、依存するPydanticという強力なライブラリによって、この型ヒントを積極的に利用し、 APIの入出力のバリデーションを行う。

### レスポンス型の定義
まず、APIのスキーマは、APIのリクエストやレスポンスの型を定義するためのもので、 データベースのスキーマとは異なることに注意する必要がある。

ルーターでは、先程定義したスキーマを利用して、APIのリクエストとレスポンスを定義する。

`GET /tasks` ではリクエストパラメータやリクエストボディは取ならいため、レスポンスだけを定義する。

レスポンスのスキーマとして、パスオペレーション関数のデコレータに response_model をセットする。

`GET /tasks` は、スキーマに定義した Task クラスを複数返しますので、リストとして定義します。ここでは、 `response_model=List[task_schema.Task]`となる。

### リクエスト型の定義
`GET /tasks` と対になる、 `POST /tasks` に対応する `create_task()` 関数を定義する。

### スキーマ
 `id`を持つ`Task`インスタンスを返却していた。
 これに対し、通常POST関数では`id`を指定せず、DBで自動的に`id`を採番することが多いです。


### スキーマ駆動開発について
さて、本章および 9章 スキーマ（Schemas） - レスポンス では、 8章 ルーター（Routers） で定義したルーターのプレースホルダーに対してリクエストとレスポンスを定義した。

しかし、まだ肝心のデータ保存やデータ読み込みが実装されていない。

ルーターとしてはそれぞれのパスオペレーションごとにたった３行のコードを書いたに過ぎないが、この時点で APIモック の役割を果たす。

すなわち、ここまで準備した段階で、フロントエンドとバックエンドのインテグレーションを開始することができる。

もちろん、条件によってリクエストやレスポンスの型が変わったり、異常ケースのすべてを扱えないことはある。


### 開発初期に与える影響
多くの他のフレームワークでは、Swagger UIのインテグレーションをサポートしていません。そのため、通常は
1. スキーマをOpen APIの形式（通常はYAML）で定義する
2. Swagger UIを提供してフロントエンド開発者に引き渡す
3. API開発に取り掛かる

というフローによってスキーマ駆動開発を実現するが、FastAPIでは、

1. API開発としてルーターとスキーマを定義し、フロントエンド開発者に引き渡す
2. 上記のルーターとスキーマをそのまま肉付けする形でAPIの機能を実装する
というずっとシンプルなステップでスキーマ駆動開発が実現できる.

### 機能修正時に与える影響
FastAPIによるスキーマ駆動開発は思ったよりも強力です。最初に開発するときだけではなく、 最初に定義したリクエストやレスポンスを変更するフロー を考えてみましょう。

通常他のフレームワークでは、

1. 最初にOpen APIで定義したスキーマを変更する
2. 変更後のSwagger UIを提供してフロントエンド開発者に引き渡す
3. APIを修正する

となるところが、FastAPIであれば

1. 動いているAPIのリクエストやレスポンスを直接変更し、同時に自動生成されたSwagger UIをフロントエンド開発者に引き渡す

一度APIを開発してしまうと、Open APIのスキーマ定義はメンテナンスされなくなり、例えばSwagger UIを提供するモックサーバーを立ち上げる方法が忘れ去られたり、そもそも壊れてしまったりすることがあるが、FastAPIはAPIインターフェイスの定義（ドキュメント）と実装が一緒になっているのでその心配がない


## データベースの接続とDBモデル（Models）

### MySQLコンテナの立ち上げ
docker-composeを利用することによってMySQLも簡単にインストールすることができる
demo-app と並列に、 demo という名前のデータベースを持つ db サービスを追加します。

```yaml
version: '3'
services:
  demo-app:
    build: .
    volumes:
      - .dockervenv:/src/.venv
      - .:/src
    ports:
      - 8000:8000  # ホストマシンのポート8000を、docker内のポート8000に接続する
  db:
    image: mysql:8.0
    platform: linux/x86_64  # M1 Macの場合必要
    environment:
      MYSQL_ALLOW_EMPTY_PASSWORD: 'yes'  # rootアカウントをパスワードなしで作成
      MYSQL_DATABASE: 'demo'  # 初期データベースとしてdemoを設定
      TZ: 'Asia/Tokyo'  # タイムゾーンを日本時間に設定
    volumes:
      - mysql_data:/var/lib/mysql
    command: --default-authentication-plugin=mysql_native_password  # MySQL8.0ではデフォルトが"caching_sha2_password"で、ドライバが非対応のため変更
    ports:
      - 33306:3306  # ホストマシンのポート33306を、docker内のポート3306に接続する
volumes:
  mysql_data:

```
上記のように`docker-compose.yaml`を用意したら
```bash
docker-compose up
```
で再度コンテナを立ち上げる。

この状態で、プロジェクトディレクトリで`docker-compose exec db mysql demmo`とすると、MySQLクラインとが実行されDBに接続できる。

(`docker-compose exec <service> <command>`で<service>で<command>を実行できる)