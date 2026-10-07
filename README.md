# local_Overleaf
ローカル環境でOverleafを実行するための手引き

# 【教員・管理者向け】ローカル Overleaf（ShareLaTeX）環境の IaC 構築・配布・固定化マニュアル

本ドキュメントは、Windows / Mac が混在する学生のPC環境に対して、**完全に同一バージョン（固定化）の日本語対応 LaTeX 環境（TeX Live scheme-full）** を Docker Compose（IaC）を用いて一発で展開するための手順書です。

---

## 🏛️ 全体構成とファイル構造

生徒へ配布する最小限の構成です。教員が事前にビルド済みイメージを Docker レジストリ（Docker Hub など）に公開することで、生徒側は **`docker-compose.yml` 1枚のみ** で環境が完成します。

```text
📁 overleaf-distribution/
├── 📄 docker-compose.yml     # 【生徒配布用】インフラ構成定義ファイル（IaC）
└── 📁 src-image/             # 【教員開発用】カスタムイメージ作成ディレクトリ
    └── 📄 Dockerfile         # 【教員開発用】TeX Live環境を固定化する定義ファイル
```

---

## 🛠️ PHASE 1：【教員作業】マルチプラットフォームイメージの作成と固定化

Windows（`amd64`）と MシリーズMac（`arm64`）の両方で動作する、日本語環境入りの Docker イメージを作成し、パブリック/プライベートレジストリへ Push します。

### 1. `src-image/Dockerfile` の作成
以下の内容で `Dockerfile` を作成します。ベースイメージのバージョンを明示的に固定します。

```dockerfile
# ベースイメージのバージョンを固定
FROM sharelatex/sharelatex:5.1.0

# タイムゾーンと環境変数の設定
ENV TZ=Asia/Tokyo
ENV PATH=/usr/local/texlive/2025/bin/x86_64-linux:/usr/local/texlive/2025/bin/aarch64-linux:$PATH

# 必要最低限の依存パッケージと日本語フォント（Noto Sans/Serif CJK）のインストール
RUN apt-get update && apt-get install -y --no-install-recommends \
    ghostscript \
    fonts-noto-cjk \
    fonts-noto-cjk-extra \
    && rm -rf /var/lib/apt/lists/*

# TeX Live のセルフアップデートとフルパッケージ（scheme-full）のインストール
RUN tlmgr update --self && \
    tlmgr install scheme-full
```

### 2. マルチプラットフォームビルド＆Push
教員の PC（Docker Desktop 起動済）のターミナルで `src-image` ディレクトリに移動し、以下のコマンドを実行します。これにより、両 CPU に対応したイメージが一発でビルドされ、Docker Hub にアップロードされます。

```bash
# 1. Docker Hub にログイン
docker login

# 2. 異なる CPU 向けにビルドするための仮想ビルダー（Buildx）を作成・有効化
docker buildx create --name classroom-builder --use
docker buildx inspect --bootstrap

# 3. 両対応ビルドとレジストリへの Push を実行（※処理には1〜2時間かかります）
docker buildx build --platform linux/amd64,linux/arm64 -t mol0711/share-overleaf-japanese:v1.0 --push .
```
上記を一行で実行してください。

---

## 📄 PHASE 2：【生徒配布用】`docker-compose.yml` の作成

PHASE 1 で作成した固定化イメージを参照する、インフラ定義ファイル（IaC）です。このファイルを生徒に配布します。

```yaml
services:
  sharelatex:
    # 教員が作成・Pushしたマルチプラットフォーム対応イメージを指定
    image: <ご自身のDockerHubユーザー名>/share-overleaf-japanese:v1.0
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
      - SHARELATEX_APP_NAME=Local Overleaf Classroom
      - SHARELATEX_MONGO_URL=mongodb://mongo/sharelatex
      - SHARELATEX_REDIS_HOST=redis
      - REDIS_HOST=redis
    volumes:
      - sharelatex_data:/var/lib/sharelatex

  mongo:
    image: mongo:4.4
    container_name: sharelatex-mongo
    restart: always
    expose:
      - "27017"
    volumes:
      - mongo_data:/data/db
    healthcheck:
      test: echo 'db.runCommand("ping").ok' | mongo localhost:27017/test --quiet
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

---

## 🚀 PHASE 3：【生徒向け作業】環境構築・利用マニュアル

学生に提示するセットアップ手順です。

### 📋 前提条件
各自の PC に **Docker Desktop** がインストールされ、起動していることを確認してください。
* **Windows ユーザー**: インストール時に「Use the WSL 2 based engine」にチェックが入っていること。
* **Mac ユーザー**: Intel Mac、Apple Silicon（M1/M2/M3/M4）Mac どちらでも構いません。

### 🏃 起動手順
1. 配布された `docker-compose.yml` を、PC 内の任意の空フォルダ（例: `overleaf`）に保存します。
2. ターミナル（Mac）または PowerShell（Windows）を開き、そのフォルダに移動します。
   ```bash
   cd path/to/overleaf
   ```
3. 以下のコマンドを実行して環境を起動します（バックグラウンドで起動します）。
   ```bash
   docker compose up -d
   ```
   *※ 初回のみイメージのダウンロードが行われますが、数分で完了します。*

### 🔑 初回アカウント作成（Launchpad）
1. コンテナの起動完了後、ブラウザを開き以下の URL にアクセスします。
   > **`http://localhost:8080/launchpad`**
2. 画面の指示に従い、各自の「メールアドレス」と「パスワード」を入力して、**最初の管理者（Admin）アカウント** を作成します。
3. 登録が完了すると、以降は `http://localhost:8080` から通常通り Overleaf を利用できるようになります。

---

## 💡 トラブルシューティング（混在環境特有の注意点）

* **エラー: `port is already allocated`（ポートの競合）**
  * **原因**: 学生の PC で、別の開発ツールやシステムがすでに `8080` ポートを使用しています。
  * **対策**: `docker-compose.yml` の `ports:` 欄を `"8081:80"` や `"9000:80"` などに変更し、再度 `docker compose up -d` を実行させてください。アクセスする URL も変更したポート（例: `http://localhost:8081`）になります。
* **コンテナを停止したい場合**
  * フォルダ内で `docker compose stop` を実行します。作成したプロジェクトやアカウントデータは Docker の Volume 領域に永続化されているため、次回 `docker compose start` で再開しても消えません。
