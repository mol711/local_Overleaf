# local_Overleaf
ローカル環境でOverleafを実行するための手引き

# 【教員・管理者向け】ローカル Overleaf（ShareLaTeX）環境の IaC 構築・配布・固定化マニュアル

本ドキュメントは、Windows / Mac が混在する学生のPC環境に対して、**完全に同一バージョン（固定化）の日本語対応 LaTeX 環境（TeX Live scheme-full）** を Docker Compose（IaC）を用いて一発で展開するための手順書です。

---

## 🏛️ 全体構成とファイル構造

生徒へ配布する最小限の構成です。教員が事前にビルド済みイメージを Docker レジストリ（Docker Hub など）に公開することで、生徒側は、次の **2ファイル** で環境を起動できます。`mongo-init-replica.js` は MongoDB を単一ノードのレプリカセットとして初期化するために必要です。ファイル名と両者の配置は変更しないでください。

```text
📁 overleaf-distribution/
├── 📄 tamplate_docker-compose.yml # 【教員用】配布用Composeのテンプレート
├── 📄 docker-compose.yml     # 【生徒配布用】教員のイメージ名を設定済みの構成定義ファイル（IaC）
├── 📄 mongo-init-replica.js  # 【生徒配布用】MongoDBレプリカセット初期化スクリプト
└── 📁 src-image/             # 【教員開発用】カスタムイメージ作成ディレクトリ
    └── 📄 Dockerfile         # 【教員開発用】TeX Live環境を固定化する定義ファイル
```

---

## 🛠️ PHASE 1：【教員作業】マルチプラットフォームイメージの作成と固定化

Windows（`amd64`）と MシリーズMac（`arm64`）の両方で動作する、日本語環境入りの Docker イメージを作成し、パブリック/プライベートレジストリへ Push します。

### 1. `src-image/Dockerfile` の作成
以下の内容で `Dockerfile` を作成します。ベースイメージのバージョンを明示的に固定します。

```dockerfile
# ベースイメージをマルチプラットフォームmanifest digestまで固定
FROM sharelatex/sharelatex:5.1.0@sha256:790b655a04ecdc07ea53276d6e3c0e5bb8cd016676402c71ea73007d2f801015

# タイムゾーンと環境変数の設定
ENV TZ=Asia/Tokyo
# sharelatex:5.1.0 には TeX Live 2024 が含まれる
ENV PATH=/usr/local/texlive/2024/bin/x86_64-linux:/usr/local/texlive/2024/bin/aarch64-linux:${PATH}

# TeX Live 2024 の最終更新を固定保存した公式アーカイブを使う
ARG TEXLIVE_REPOSITORY=https://ftp.math.utah.edu/pub/tex/historic/systems/texlive/2024/tlnet-final

# 必要最低限の依存パッケージと日本語フォント（Noto Sans/Serif CJK）のインストール
RUN apt-get update && apt-get install -y --no-install-recommends \
    ghostscript \
    fonts-noto-cjk \
    fonts-noto-cjk-extra \
    && rm -rf /var/lib/apt/lists/*

# TeX Live 2024 の最終状態へ更新してからフルパッケージを導入し、formatを検証する
RUN tlmgr option repository "${TEXLIVE_REPOSITORY}" && \
    tlmgr update --self --all && \
    tlmgr install scheme-full && \
    kpsewhich utf8mex.ini && \
    fmtutil-sys --all
```

> **固定化について**: `sharelatex:5.1.0` のTeX Liveは2024です。通常の`https://mirror.ctan.org/systems/texlive/tlnet`は更新され続けるため、別年度のパッケージ索引と2024の本体が混ざり、チェックサム不一致やformat生成失敗を起こします。この定義ではTeX Live 2024の最終アーカイブ（`tlnet-final`）だけを参照し、`scheme-full`を含む同一のパッケージ集合を両CPU向けに作成します。

### 2. マルチプラットフォームビルド＆Push
教員の PC（Docker Desktop 起動済）のターミナルで `src-image` ディレクトリに移動し、以下のコマンドを実行します。`TEACHER_DOCKERHUB_USERNAME` は、教員が使用する Docker Hub ユーザー名に置き換えてください。これにより、両 CPU に対応したイメージが一発でビルドされ、Docker Hub にアップロードされます。

```bash
# 1. Docker Hub にログイン
docker login

# 2. 異なる CPU 向けにビルドするための仮想ビルダー（Buildx）を作成・有効化
docker buildx create --name classroom-builder --use
docker buildx inspect --bootstrap

# 3. 両対応ビルドとレジストリへの Push を実行（※処理には1〜2時間かかります）
docker buildx build --platform linux/amd64,linux/arm64 -t TEACHER_DOCKERHUB_USERNAME/share-overleaf-japanese:v1.0 --push .
```
上記の3.を一行で実行してください。

---

## 📄 PHASE 2：【生徒配布用】起動ファイルの作成

PHASE 1 で作成した固定化イメージを参照するインフラ定義ファイル（IaC）と、MongoDB初期化スクリプトです。**`docker-compose.yml` と `mongo-init-replica.js` の2ファイルを必ず同じフォルダに置いて配布**します。

`tamplate_docker-compose.yml` は教員用のテンプレートです。教員はこれを `docker-compose.yml` としてコピーし、`<教員から指定されたDockerHubユーザー名>` を実際の Docker Hub ユーザー名へ置き換えてから学生へ配布します。テンプレートのままでは起動できません。イメージを非公開にする場合は、受講者にイメージ閲覧権限を付与し、各自に Docker Hub の認証情報で `docker login` を実行させます。Docker Hub のパスワードやアクセストークンを YAML・README・配布物に書かないでください。

```yaml
services:
  sharelatex:
    # <教員から指定されたDockerHubユーザー名> を実際の値に置き換える
    image: <教員から指定されたDockerHubユーザー名>/share-overleaf-japanese:v1.0
    container_name: sharelatex-classroom
    restart: always
    # 学生のPCで他のアプリと衝突を避けるため、Webアクセスポートを 8080 に設定
    ports:
      - "8080:80"
    links:
      - mongo
      - redis
    depends_on:
      mongo:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      - OVERLEAF_APP_NAME=Local Overleaf Classroom
      - OVERLEAF_MONGO_URL=mongodb://mongo/sharelatex
      - OVERLEAF_REDIS_HOST=redis
      - REDIS_HOST=redis
    volumes:
      - sharelatex_data:/var/lib/overleaf

  mongo:
    image: mongo:5.0
    container_name: sharelatex-mongo
    restart: always
    command: ["mongod", "--replSet", "rs0", "--bind_ip_all"]
    expose:
      - "27017"
    volumes:
      - mongo_data:/data/db
      - ./mongo-init-replica.js:/docker-entrypoint-initdb.d/mongo-init-replica.js:ro
    healthcheck:
      test: ["CMD-SHELL", "mongosh --quiet --eval 'db.hello().isWritablePrimary' | grep true"]
      interval: 10s
      timeout: 10s
      retries: 5

  redis:
    image: redis:6.2
    container_name: sharelatex-redis
    restart: always
    expose:
      - "6379"
    volumes:
      - redis_data:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 10s
      retries: 5

volumes:
  sharelatex_data:
  mongo_data:
  redis_data:
```

同じフォルダに、MongoDBをトランザクション対応の単一ノードレプリカセットとして初期化する`mongo-init-replica.js`を置きます。

```javascript
try {
  rs.status()
} catch (error) {
  rs.initiate({
    _id: "rs0",
    members: [{ _id: 0, host: "mongo:27017" }],
  })
}
```

---

## 🚀 PHASE 3：【生徒向け作業】環境構築・利用マニュアル

学生に提示するセットアップ手順です。
教員は配布する前に`template_docker-compose.yml`を`docker-compose.yml`に変更し、内部の`image`で自分のDockerhubのレポジトリが指定されていることを確認する。

### 📋 前提条件
各自の PC に **Docker Desktop** がインストールされ、起動していることを確認してください。
* **Windows ユーザー**: インストール時に「Use the WSL 2 based engine」にチェックが入っていること。
* **Mac ユーザー**: Intel Mac、Apple Silicon（M1/M2/M3/M4）Mac どちらでも構いません。

### 🏃 起動手順
1. 教員から配布された `docker-compose.yml` と `mongo-init-replica.js` を、PC 内の任意の空フォルダ（例: `overleaf`）に保存します。`mongo-init-replica.js` を省略・改名すると起動できません。
2. ターミナル（Mac）または PowerShell（Windows）を開き、そのフォルダに移動します。
   ```bash
   cd path/to/overleaf
   ```
3. 以下のコマンドを実行して環境を起動します（バックグラウンドで起動します）。
   ```bash
   docker compose up -d
   ```
   *※ 初回のみイメージのダウンロードが行われますが、数分で完了します。*

### 🔑 初回アカウント作成（管理者）
1. コンテナの起動完了後、ブラウザを開き以下の URL にアクセスします。
   > **`http://localhost:8080/launchpad`**
2. 画面の指示に従い、各自の「メールアドレス」と「パスワード」を入力して、**最初の管理者（Admin）アカウント** を作成します。
   * この構成はローカルで動作し、外部のメール認証やインターネット上のOverleafアカウントとは連携しません。メールアドレスには `admin@example.local` のような実在しない値を入力して構いません。パスワードも外部サービスで使用しているものを使い回す必要はありません。
3. 「ログインページへ進む」リンクを選ぶか、`http://localhost:8080/login` を開きます。
4. 手順2で設定したメールアドレスとパスワードでログインします。Welcome 画面が表示されたら、画面下部のボタンを選んで Overleaf を開始します。

### 👥 一般ユーザーを追加してログインさせる（管理者向け）

この操作は、同じ Overleaf 環境を複数人で使う場合だけ必要です。各学生が自分のPCにこの構成を起動する運用では、学生ごとに上の「初回アカウント作成」を行えばよく、管理者がユーザーを追加する必要はありません。

1. 管理者アカウントで `http://localhost:8080/login` にログインします。
2. `http://localhost:8080/admin/register` を開きます。
3. 追加するユーザーのメールアドレスを入力し、登録します。入力は小文字で統一してください。
4. この構成にはメール送信設定がないため、登録直後に画面に表示される**パスワード設定用URL**をコピーします。
5. そのURLを、追加した本人に安全な方法で渡します。URLを開いた本人はパスワードを設定し、続けて `http://localhost:8080/login` からメールアドレスと設定したパスワードでログインします。

> **注意**: `localhost` は、そのURLを開くPC自身を指します。同一PC上で複数アカウントを作る用途では上記のURLをそのまま使えます。別のPCから同じOverleaf環境を利用させる場合は、ホスト名・ネットワーク公開・アクセス制御を別途設計してから、配布するURLをその接続先に置き換えてください。

---

## 💡 トラブルシューティング（混在環境特有の注意点）

* **エラー: `port is already allocated`（ポートの競合）**
  * **原因**: 学生の PC で、別の開発ツールやシステムがすでに `8080` ポートを使用しています。
  * **対策**: `docker-compose.yml` の `ports:` 欄を `"8081:80"` や `"9000:80"` などに変更し、再度 `docker compose up -d` を実行させてください。アクセスする URL も変更したポート（例: `http://localhost:8081`）になります。
* **コンテナを停止したい場合**
  * フォルダ内で `docker compose stop` を実行します。作成したプロジェクトやアカウントデータは Docker の Volume 領域に永続化されているため、次回 `docker compose start` で再開しても消えません。
