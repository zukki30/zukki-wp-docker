# 実装計画書: WordPress Docker環境構築

## ドキュメント情報
- **spec名**: wordpress-docker-setup
- **作成日**: 2026-02-17
- **最終更新日**: 2026-02-17
- **ステータス**: レビュー待ち
- **バージョン**: 1.0

---

## 1. 実装概要

### 1.1 プロジェクトの目的

Docker ComposeでWordPress開発環境を構築し、以下の特徴を持つシステムを実装します。

- 環境変数のみで構成を柔軟に切り替え可能（Webサーバー、データベース、PHPバージョン等）
- データ永続化によるコンテナ再作成時のデータ保持
- デバッグモード有効化による開発効率向上
- phpMyAdminによるデータベース管理機能（オプション）

### 1.2 実装の優先順位と依存関係

#### 1.2.1 実装フェーズ

| フェーズ | 内容 | 優先度 | 依存関係 |
|---------|------|--------|---------|
| Phase 1 | 基本ファイル作成（.gitignore, .env.example） | 最高 | なし |
| Phase 2 | docker-compose.yml実装（Apache + MySQL基本構成） | 最高 | Phase 1 |
| Phase 3 | Nginx設定ファイル作成 | 高 | Phase 2 |
| Phase 4 | phpMyAdmin統合 | 中 | Phase 2 |
| Phase 5 | README.md作成 | 高 | Phase 2, 3, 4 |
| Phase 6 | 動作検証・テスト | 最高 | Phase 1-5 |

#### 1.2.2 依存関係図

```
Phase 1 (基本ファイル)
    ↓
Phase 2 (docker-compose.yml基本構成)
    ↓ ←─────────────┐
    ├─ Phase 3 (Nginx) │
    │                   │
    └─ Phase 4 (phpMyAdmin)
              ↓
    Phase 5 (README.md)
              ↓
    Phase 6 (動作検証)
```

### 1.3 想定される実装期間と工数

| タスク | 作業時間 | 難易度 |
|-------|---------|--------|
| .gitignore作成 | 10分 | 低 |
| .env.example作成 | 30分 | 低 |
| docker-compose.yml作成（Apache構成） | 1時間 | 中 |
| docker-compose.yml拡張（Nginx対応） | 1時間 | 中 |
| nginx.conf作成 | 30分 | 中 |
| phpMyAdmin統合 | 30分 | 低 |
| README.md作成 | 1時間 | 低 |
| 動作検証（4パターン × テスト） | 2時間 | 中 |
| **合計** | **約6.5時間** | - |

---

## 2. 技術スタック

### 2.1 使用する言語・フレームワーク・ライブラリ

#### 2.1.1 コンテナイメージ

| コンポーネント | イメージ | バージョン範囲 | 用途 |
|--------------|---------|--------------|------|
| WordPress (Apache) | `wordpress:php{VERSION}-apache` | PHP 8.1-8.3 | WordPress本体（Apache使用時） |
| WordPress (FPM) | `wordpress:php{VERSION}-fpm` | PHP 8.1-8.3 | WordPress本体（Nginx使用時） |
| MySQL | `mysql:{VERSION}` | 8.0, 8.4 | データベース |
| MariaDB | `mariadb:{VERSION}` | 10.11, 11.4 | データベース（代替） |
| Nginx | `nginx:alpine` | latest | Webサーバー（FPM使用時） |
| phpMyAdmin | `phpmyadmin:{VERSION}` | latest, 5.2, 5.1 | データベース管理UI |

#### 2.1.2 設定ファイル形式

| ファイル | フォーマット | 用途 |
|---------|------------|------|
| docker-compose.yml | YAML | Docker Compose設定 |
| .env | KEY=VALUE | 環境変数定義 |
| nginx.conf | Nginx設定形式 | Nginx設定 |
| .gitignore | テキスト | Git除外設定 |
| README.md | Markdown | ドキュメント |

### 2.2 開発環境とツール

#### 2.2.1 必須ツール

| ツール | 最小バージョン | 確認コマンド |
|-------|--------------|------------|
| Docker | 20.10.0+ | `docker --version` |
| Docker Compose | 2.0.0+ | `docker-compose --version` |

#### 2.2.2 推奨ツール

| ツール | 用途 |
|-------|------|
| Git | バージョン管理 |
| テキストエディタ | 設定ファイル編集（VS Code, vim等） |
| ブラウザ | WordPress/phpMyAdmin アクセス |

### 2.3 外部サービスやAPIの利用

本プロジェクトでは外部サービスやAPIは使用しません。すべてローカル環境で完結します。

---

## 3. ファイル構成

### 3.1 作成・修正が必要なファイルの一覧

| No. | ファイルパス | 種別 | 優先度 | 備考 |
|----|-------------|------|--------|------|
| 1 | `.gitignore` | 新規作成 | 最高 | Git除外設定 |
| 2 | `.env.example` | 新規作成 | 最高 | 環境変数テンプレート |
| 3 | `docker-compose.yml` | 新規作成 | 最高 | メイン構成ファイル |
| 4 | `nginx/nginx.conf` | 新規作成 | 高 | Nginx設定（FPM使用時） |
| 5 | `README.md` | 新規作成 | 高 | セットアップ手順 |
| 6 | `wordpress/wp-content/.gitkeep` | 新規作成 | 中 | ディレクトリ保持用 |

### 3.2 ディレクトリ構造

```
zukki-wp-docker/
├── .gitignore                    # Git除外設定
├── .env.example                  # 環境変数テンプレート
├── .env                          # 環境変数（ユーザーが作成、Git管理外）
├── docker-compose.yml            # Docker Compose設定
├── README.md                     # セットアップ手順
├── nginx/
│   └── nginx.conf               # Nginx設定ファイル
└── wordpress/
    └── wp-content/              # WordPressファイル（自動生成、Git管理外）
        ├── .gitkeep            # ディレクトリ保持用
        ├── themes/             # テーマ（自動生成）
        ├── plugins/            # プラグイン（自動生成）
        ├── uploads/            # アップロードファイル（自動生成）
        └── debug.log           # デバッグログ（自動生成）
```

### 3.3 各ファイルの役割と責務

#### 3.3.1 .gitignore
- **役割**: Git管理から除外するファイルを定義
- **責務**: 機密情報やローカル生成ファイルの保護

#### 3.3.2 .env.example
- **役割**: 環境変数のテンプレートとドキュメント
- **責務**: ユーザーが.envファイルを作成する際のガイド

#### 3.3.3 docker-compose.yml
- **役割**: Docker Composeのメイン設定ファイル
- **責務**: コンテナ定義、ネットワーク、ボリューム管理

#### 3.3.4 nginx/nginx.conf
- **役割**: Nginx設定ファイル（WP_SERVER=fpm時に使用）
- **責務**: FastCGI設定、静的ファイル配信、WordPress用ルーティング

#### 3.3.5 README.md
- **役割**: プロジェクトのドキュメント
- **責務**: セットアップ手順、使用方法、トラブルシューティング

---

## 4. 実装手順

### 4.1 Phase 1: 基本ファイル作成

#### 4.1.1 .gitignore作成

**目的**: Git管理から除外すべきファイルを定義する

**実装内容**:
```gitignore
# 環境設定ファイル（機密情報を含む）
.env

# WordPress データ（ボリュームマウントで生成される）
wordpress/wp-content/*
!wordpress/wp-content/.gitkeep

# データベースバックアップ
*.sql
*.sql.gz

# ログファイル
*.log

# macOS
.DS_Store

# Windows
Thumbs.db

# IDE
.vscode/
.idea/
*.swp
*.swo
*~
```

**検証方法**:
- `.gitignore`ファイルが作成されていること
- 各除外パターンが適切にコメントされていること

#### 4.1.2 .env.example作成

**目的**: 環境変数のテンプレートを提供する

**実装内容**:
```bash
# ============================================================
# WordPress Docker環境 設定ファイル
# ============================================================
# このファイルをコピーして .env ファイルを作成してください
#   cp .env.example .env
# 本番環境では使用しないでください（開発環境専用）
# ============================================================

# ------------------------------------------------------------
# WordPress設定
# ------------------------------------------------------------
# WordPressバージョン（例: latest, 6.4, 6.3）
WP_VERSION=latest

# PHPバージョン（例: 8.1, 8.2, 8.3）
PHP_VERSION=8.2

# Webサーバー種類（apache または fpm）
# - apache: Apache + mod_php（シンプル、設定不要）
# - fpm: Nginx + PHP-FPM（高パフォーマンス、nginx設定必要）
WP_SERVER=apache

# WordPressアクセスポート（デフォルト: 8080）
# http://localhost:${WP_PORT} でアクセスできます
WP_PORT=8080

# ------------------------------------------------------------
# データベース設定
# ------------------------------------------------------------
# データベース種類（mysql または mariadb）
DB_TYPE=mysql

# データベースバージョン
# - MySQL: 8.0, 8.4
# - MariaDB: 10.11, 11.4
DB_VERSION=8.0

# データベース名
MYSQL_DATABASE=wordpress

# データベースユーザー名
MYSQL_USER=wordpress

# データベースパスワード
# 重要: 本番環境では必ず変更してください
MYSQL_PASSWORD=wordpress

# データベースrootパスワード
# 重要: 本番環境では必ず変更してください
MYSQL_ROOT_PASSWORD=rootpassword

# ------------------------------------------------------------
# phpMyAdmin設定（データベース管理ツール）
# ------------------------------------------------------------
# phpMyAdminを有効化するか（true または false）
# - true: phpMyAdminコンテナが起動します
# - false: phpMyAdminコンテナは起動しません（デフォルト）
PHPMYADMIN_ENABLED=false

# phpMyAdminバージョン（例: latest, 5.2, 5.1）
PHPMYADMIN_VERSION=latest

# phpMyAdminアクセスポート（デフォルト: 8081）
# http://localhost:${PHPMYADMIN_PORT} でアクセスできます
PHPMYADMIN_PORT=8081

# ------------------------------------------------------------
# 注意事項
# ------------------------------------------------------------
# 1. このファイルには機密情報が含まれるため、Gitにコミットしないでください
# 2. 本設定はローカル開発環境専用です
# 3. デバッグモードが有効になっています（本番環境では無効化してください）
# 4. デフォルトパスワードは必ず変更してください
```

**検証方法**:
- `.env.example`ファイルが作成されていること
- すべての必要な環境変数が記載されていること
- 各変数に説明コメントが付いていること

#### 4.1.3 wordpress/wp-content/.gitkeep作成

**目的**: 空のディレクトリをGit管理下に置く

**実装内容**:
```bash
mkdir -p wordpress/wp-content
touch wordpress/wp-content/.gitkeep
```

**検証方法**:
- `wordpress/wp-content/`ディレクトリが存在すること
- `.gitkeep`ファイルが存在すること

---

### 4.2 Phase 2: docker-compose.yml実装

#### 4.2.1 基本構造の実装

**目的**: Apache + MySQL構成を基本として実装する

**実装内容**:

```yaml
version: '3.8'

services:
  # ------------------------------------------------------------
  # WordPress サービス
  # ------------------------------------------------------------
  wordpress:
    image: wordpress:php${PHP_VERSION:-8.2}-${WP_SERVER:-apache}
    container_name: wordpress-${WP_SERVER:-apache}
    restart: unless-stopped
    ports:
      # Apache使用時のみポート公開（Nginx使用時は公開しない）
      - "${WP_PORT:-8080}:80"
    environment:
      # データベース接続設定
      WORDPRESS_DB_HOST: db:3306
      WORDPRESS_DB_NAME: ${MYSQL_DATABASE:-wordpress}
      WORDPRESS_DB_USER: ${MYSQL_USER:-wordpress}
      WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:-wordpress}
      # デバッグ設定（開発環境用）
      WORDPRESS_DEBUG: 1
      WORDPRESS_DEBUG_LOG: 1
      WORDPRESS_DEBUG_DISPLAY: 1
    volumes:
      # wp-content をホストにマウント（テーマ・プラグイン開発用）
      - ./wordpress/wp-content:/var/www/html/wp-content
    depends_on:
      db:
        condition: service_healthy
    networks:
      - wordpress-network

  # ------------------------------------------------------------
  # データベースサービス
  # ------------------------------------------------------------
  db:
    image: ${DB_TYPE:-mysql}:${DB_VERSION:-8.0}
    container_name: wordpress-db
    restart: unless-stopped
    environment:
      MYSQL_DATABASE: ${MYSQL_DATABASE:-wordpress}
      MYSQL_USER: ${MYSQL_USER:-wordpress}
      MYSQL_PASSWORD: ${MYSQL_PASSWORD:-wordpress}
      MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD:-rootpassword}
    volumes:
      # データベースデータを名前付きボリュームで永続化
      - db_data:/var/lib/mysql
    networks:
      - wordpress-network
    healthcheck:
      test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-rootpassword}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # ------------------------------------------------------------
  # Nginx サービス（WP_SERVER=fpm時のみ起動）
  # ------------------------------------------------------------
  nginx:
    image: nginx:alpine
    container_name: wordpress-nginx
    restart: unless-stopped
    profiles:
      - nginx
    ports:
      - "${WP_PORT:-8080}:80"
    volumes:
      # Nginx設定ファイル（読み取り専用）
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
      # wp-content を読み取り専用でマウント（静的ファイル配信用）
      - ./wordpress/wp-content:/var/www/html/wp-content:ro
    depends_on:
      - wordpress
    networks:
      - wordpress-network

  # ------------------------------------------------------------
  # phpMyAdmin サービス（PHPMYADMIN_ENABLED=true時のみ起動）
  # ------------------------------------------------------------
  phpmyadmin:
    image: phpmyadmin:${PHPMYADMIN_VERSION:-latest}
    container_name: wordpress-phpmyadmin
    restart: unless-stopped
    profiles:
      - phpmyadmin
    ports:
      - "${PHPMYADMIN_PORT:-8081}:80"
    environment:
      PMA_HOST: db
      PMA_USER: ${MYSQL_USER:-wordpress}
      PMA_PASSWORD: ${MYSQL_PASSWORD:-wordpress}
    depends_on:
      - db
    networks:
      - wordpress-network

# ------------------------------------------------------------
# ネットワーク定義
# ------------------------------------------------------------
networks:
  wordpress-network:
    driver: bridge

# ------------------------------------------------------------
# ボリューム定義
# ------------------------------------------------------------
volumes:
  db_data:
    driver: local
```

**実装のポイント**:

1. **環境変数のデフォルト値設定**
   - 形式: `${変数名:-デフォルト値}`
   - `.env`ファイルがない場合でも動作する

2. **イメージタグの動的構成**
   - `wordpress:php${PHP_VERSION}-${WP_SERVER}`
   - Apache使用時: `wordpress:php8.2-apache`
   - FPM使用時: `wordpress:php8.2-fpm`

3. **profiles機能の活用**
   - `nginx`: Nginx使用時のみ起動
   - `phpmyadmin`: phpMyAdmin有効時のみ起動

4. **ヘルスチェック設定**
   - データベースの起動完了を確認
   - WordPressが接続エラーで失敗しないようにする

5. **ポート公開の条件分岐**
   - Apache使用時: `wordpress`コンテナがポート公開
   - Nginx使用時: `nginx`コンテナがポート公開
   - ※docker-compose.ymlでは両方定義し、起動時にprofileで制御

**検証方法**:
- YAMLの構文が正しいこと: `docker-compose config`
- すべての環境変数が参照されていること
- デフォルト値が設定されていること

#### 4.2.2 Apache/FPM切り替えの実装

**課題**: 同じ`wordpress`サービスでApache/FPMを切り替えるが、ポート公開の有無が異なる

**解決策**: docker-compose.override.ymlを使用する

**実装方針**:
1. **基本**: `docker-compose.yml`にApache構成を記述（デフォルト）
2. **拡張**: Nginx使用時はprofileで`nginx`コンテナを起動
3. **ポート**: Apache使用時は`wordpress`がポート公開、Nginx使用時は`nginx`がポート公開

**起動コマンド**:
```bash
# Apache使用時（デフォルト）
docker-compose up -d

# Nginx使用時
docker-compose --profile nginx up -d

# phpMyAdmin有効時
docker-compose --profile phpmyadmin up -d

# Nginx + phpMyAdmin
docker-compose --profile nginx --profile phpmyadmin up -d
```

**注意点**:
- `WP_SERVER=fpm`を設定した場合、必ず`--profile nginx`を指定する
- `WP_SERVER=apache`の場合、`--profile nginx`は指定しない

---

### 4.3 Phase 3: Nginx設定ファイル作成

#### 4.3.1 nginx/nginx.conf作成

**目的**: Nginx + PHP-FPM構成でWordPressを動作させる

**実装内容**:

```nginx
# ============================================================
# Nginx設定ファイル（WordPress + PHP-FPM用）
# ============================================================
# WP_SERVER=fpm の場合にのみ使用されます
# ============================================================

events {
    worker_connections 1024;
}

http {
    # ------------------------------------------------------------
    # 基本設定
    # ------------------------------------------------------------
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    # ログ設定
    access_log /var/log/nginx/access.log;
    error_log /var/log/nginx/error.log;

    # パフォーマンス設定
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # クライアントボディサイズ制限（メディアアップロード用）
    client_max_body_size 64M;

    # ------------------------------------------------------------
    # WordPress サーバー設定
    # ------------------------------------------------------------
    server {
        listen 80;
        server_name localhost;

        root /var/www/html;
        index index.php index.html index.htm;

        # ログ設定（サーバー単位）
        access_log /var/log/nginx/wordpress_access.log;
        error_log /var/log/nginx/wordpress_error.log;

        # ------------------------------------------------------------
        # WordPressパーマリンク設定
        # ------------------------------------------------------------
        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        # ------------------------------------------------------------
        # PHP-FPM設定
        # ------------------------------------------------------------
        location ~ \.php$ {
            # セキュリティ: 存在しないPHPファイルへのアクセスを拒否
            try_files $uri =404;

            # FastCGI設定
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass wordpress:9000;
            fastcgi_index index.php;

            # FastCGIパラメータ
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;

            # バッファ設定
            fastcgi_buffers 8 16k;
            fastcgi_buffer_size 32k;

            # タイムアウト設定
            fastcgi_read_timeout 300;
        }

        # ------------------------------------------------------------
        # 静的ファイルのキャッシュ設定
        # ------------------------------------------------------------
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires max;
            add_header Cache-Control "public, immutable";
            log_not_found off;
            access_log off;
        }

        # ------------------------------------------------------------
        # アクセス禁止設定
        # ------------------------------------------------------------
        # .htaccessファイルへのアクセス拒否
        location ~ /\.ht {
            deny all;
        }

        # .gitディレクトリへのアクセス拒否
        location ~ /\.git {
            deny all;
        }

        # wp-config.phpへの直接アクセス拒否
        location = /wp-config.php {
            deny all;
        }

        # xmlrpc.phpへのアクセス拒否（DDoS対策）
        location = /xmlrpc.php {
            deny all;
        }
    }
}
```

**実装のポイント**:

1. **FastCGI設定**
   - `fastcgi_pass wordpress:9000`: PHP-FPMコンテナへの接続
   - `SCRIPT_FILENAME`: 実行するPHPファイルのフルパス
   - `PATH_INFO`: WordPress用のパス情報

2. **パフォーマンス最適化**
   - `client_max_body_size 64M`: 要件NFR-PERF-003に対応
   - 静的ファイルのキャッシュ設定
   - FastCGIバッファ設定

3. **セキュリティ設定**
   - `.htaccess`, `.git`へのアクセス拒否
   - `wp-config.php`への直接アクセス拒否
   - `xmlrpc.php`へのアクセス拒否（DDoS対策）

**検証方法**:
- Nginx設定の構文チェック: `docker-compose run --rm nginx nginx -t`
- nginx.confファイルが存在すること
- すべての設定項目にコメントが記載されていること

#### 4.3.2 nginxディレクトリの作成

**実装内容**:
```bash
mkdir -p nginx
# nginx.confを上記内容で作成
```

---

### 4.4 Phase 4: phpMyAdmin統合

#### 4.4.1 docker-compose.ymlへのphpMyAdmin追加

**実装内容**: Phase 2で既に実装済み

**検証項目**:
- `phpmyadmin`サービスが定義されていること
- `profiles: ["phpmyadmin"]`が設定されていること
- 環境変数`PMA_HOST`, `PMA_USER`, `PMA_PASSWORD`が設定されていること

---

### 4.5 Phase 5: README.md作成

#### 4.5.1 README.md実装

**目的**: ユーザーがセットアップから運用まで迷わず実行できるドキュメントを提供する

**実装内容**:

```markdown
# WordPress Docker環境

Docker ComposeでWordPress開発環境を構築するプロジェクトです。

## 特徴

- 環境変数のみで構成を柔軟に切り替え可能
  - Webサーバー: Apache / Nginx
  - データベース: MySQL / MariaDB
  - PHPバージョン: 8.1 / 8.2 / 8.3
  - phpMyAdmin: 有効 / 無効
- データ永続化（コンテナ再作成後もデータ保持）
- デバッグモード有効化（開発効率向上）
- phpMyAdminによるデータベース管理（オプション）

## 前提条件

以下がインストールされていることを確認してください。

- Docker: 20.10.0以上
- Docker Compose: 2.0.0以上

確認コマンド:
```bash
docker --version
docker-compose --version
```

## セットアップ手順

### 1. リポジトリのクローン

```bash
git clone <repository-url>
cd zukki-wp-docker
```

### 2. 環境変数ファイルの作成

```bash
cp .env.example .env
```

### 3. 環境変数の編集（オプション）

`.env`ファイルを編集して、必要に応じて設定を変更します。

```bash
# エディタで編集
vim .env
```

主な設定項目:
- `WP_SERVER`: `apache`（デフォルト）または `fpm`（Nginx使用）
- `DB_TYPE`: `mysql`（デフォルト）または `mariadb`
- `PHP_VERSION`: `8.2`（デフォルト）、`8.1`, `8.3`
- `PHPMYADMIN_ENABLED`: `false`（デフォルト）または `true`

### 4. コンテナの起動

**パターン1: Apache + MySQL（デフォルト）**
```bash
docker-compose up -d
```

**パターン2: Apache + MySQL + phpMyAdmin**
```bash
docker-compose --profile phpmyadmin up -d
```

**パターン3: Nginx + MySQL**
```bash
# .envでWP_SERVER=fpmに設定
docker-compose --profile nginx up -d
```

**パターン4: Nginx + MariaDB + phpMyAdmin**
```bash
# .envでWP_SERVER=fpm、DB_TYPE=mariadbに設定
docker-compose --profile nginx --profile phpmyadmin up -d
```

### 5. WordPressセットアップ

ブラウザで以下にアクセスします。

```
http://localhost:8080
```

WordPressのインストール画面が表示されたら、画面の指示に従ってセットアップを完了します。

### 6. phpMyAdminへのアクセス（有効時のみ）

```
http://localhost:8081
```

自動ログインされるため、ユーザー名・パスワードの入力は不要です。

## 構成パターン

### Pattern 1: Apache + MySQL

**用途**: 標準的なLAMP環境

**.env設定**:
```bash
WP_SERVER=apache
DB_TYPE=mysql
DB_VERSION=8.0
PHPMYADMIN_ENABLED=false
```

**起動コマンド**:
```bash
docker-compose up -d
```

### Pattern 2: Apache + MariaDB

**用途**: MariaDB使用環境

**.env設定**:
```bash
WP_SERVER=apache
DB_TYPE=mariadb
DB_VERSION=11.4
PHPMYADMIN_ENABLED=false
```

**起動コマンド**:
```bash
docker-compose up -d
```

### Pattern 3: Nginx + MySQL

**用途**: 高パフォーマンス環境

**.env設定**:
```bash
WP_SERVER=fpm
DB_TYPE=mysql
DB_VERSION=8.0
PHPMYADMIN_ENABLED=false
```

**起動コマンド**:
```bash
docker-compose --profile nginx up -d
```

### Pattern 4: Nginx + MariaDB + phpMyAdmin

**用途**: フル機能環境

**.env設定**:
```bash
WP_SERVER=fpm
DB_TYPE=mariadb
DB_VERSION=11.4
PHPMYADMIN_ENABLED=true
```

**起動コマンド**:
```bash
docker-compose --profile nginx --profile phpmyadmin up -d
```

## 基本操作

### コンテナの起動

```bash
# Apache使用時
docker-compose up -d

# Nginx使用時
docker-compose --profile nginx up -d

# phpMyAdmin有効時
docker-compose --profile phpmyadmin up -d

# Nginx + phpMyAdmin
docker-compose --profile nginx --profile phpmyadmin up -d
```

### コンテナの停止

```bash
docker-compose stop
```

### コンテナの再起動

```bash
docker-compose restart
```

### コンテナの削除（データは保持）

```bash
docker-compose down
```

### コンテナとデータの完全削除

**警告**: すべてのデータが削除されます。

```bash
docker-compose down -v
```

### ログの確認

```bash
# すべてのコンテナのログ
docker-compose logs

# 特定のコンテナのログ
docker-compose logs wordpress
docker-compose logs db

# リアルタイムでログを監視
docker-compose logs -f
```

### コンテナの状態確認

```bash
docker-compose ps
```

## データ管理

### データベースのバックアップ

```bash
docker-compose exec db mysqldump -u wordpress -pwordpress wordpress > backup.sql
```

### データベースのリストア

```bash
docker-compose exec -T db mysql -u wordpress -pwordpress wordpress < backup.sql
```

### wp-contentのバックアップ

```bash
tar -czf wp-content-backup.tar.gz wordpress/wp-content/
```

### wp-contentのリストア

```bash
tar -xzf wp-content-backup.tar.gz
```

## バージョン更新

### WordPressバージョンの更新

```bash
# 1. .envでバージョン変更
# WP_VERSION=latest → WP_VERSION=6.5

# 2. コンテナ再作成
docker-compose up -d --force-recreate wordpress

# 3. ブラウザでWordPressにアクセスし、データベース更新を実行
```

### データベースバージョンの更新

```bash
# 1. データベースバックアップ
docker-compose exec db mysqldump -u wordpress -pwordpress wordpress > backup.sql

# 2. .envでバージョン変更
# DB_VERSION=8.0 → DB_VERSION=8.4

# 3. コンテナ再作成
docker-compose up -d --force-recreate db

# 4. 動作確認
```

## トラブルシューティング

### ポート競合エラー

**エラー例**:
```
Error starting userland proxy: listen tcp4 0.0.0.0:8080: bind: address already in use
```

**対処方法**:
1. `.env`ファイルで別のポートに変更
   ```bash
   WP_PORT=8000
   ```
2. コンテナを再起動
   ```bash
   docker-compose up -d
   ```

### データベース接続エラー

**エラー例**:
```
Error establishing a database connection
```

**対処方法**:
1. データベースコンテナのログを確認
   ```bash
   docker-compose logs db
   ```
2. データベースコンテナが起動しているか確認
   ```bash
   docker-compose ps
   ```
3. データベースのヘルスチェック状態を確認
   ```bash
   docker-compose ps
   # STATE列が "healthy" になっていることを確認
   ```

### パーミッションエラー

**エラー例**:
```
Warning: Unable to create directory wp-content/uploads/
```

**対処方法**:
```bash
chmod -R 755 wordpress/wp-content
```

### Nginx起動エラー

**対処方法**:
1. nginx.confの構文チェック
   ```bash
   docker-compose run --rm nginx nginx -t
   ```
2. Nginxコンテナのログを確認
   ```bash
   docker-compose logs nginx
   ```

### 環境変数が反映されない

**対処方法**:
1. `.env`ファイルが存在するか確認
2. 環境変数が正しく設定されているか確認
   ```bash
   docker-compose config
   ```
3. コンテナを完全に再作成
   ```bash
   docker-compose down
   docker-compose up -d
   ```

### phpMyAdminにアクセスできない

**確認項目**:
1. `PHPMYADMIN_ENABLED=true`に設定されているか
2. `--profile phpmyadmin`オプションを指定して起動したか
3. phpMyAdminコンテナが起動しているか
   ```bash
   docker-compose ps
   ```

## 開発時の注意事項

### デバッグモードが有効

本環境ではWordPressのデバッグモードが有効になっています。

- エラーが画面に表示されます
- `wordpress/wp-content/debug.log`にログが記録されます

本番環境では必ずデバッグモードを無効化してください。

### デフォルトパスワードの変更

`.env`ファイルのデフォルトパスワードは開発環境用です。本番環境では必ず変更してください。

### ローカル環境専用

本設定はローカル開発環境専用です。本番環境では使用しないでください。

## ファイル構成

```
zukki-wp-docker/
├── .gitignore                    # Git除外設定
├── .env.example                  # 環境変数テンプレート
├── .env                          # 環境変数（Git管理外）
├── docker-compose.yml            # Docker Compose設定
├── README.md                     # このファイル
├── nginx/
│   └── nginx.conf               # Nginx設定ファイル
└── wordpress/
    └── wp-content/              # WordPressファイル（Git管理外）
        ├── themes/              # テーマ
        ├── plugins/             # プラグイン
        ├── uploads/             # アップロードファイル
        └── debug.log            # デバッグログ
```

## ライセンス

本プロジェクトはローカル開発環境用です。

## 参考情報

- [WordPress公式Dockerイメージ](https://hub.docker.com/_/wordpress)
- [MySQL公式Dockerイメージ](https://hub.docker.com/_/mysql)
- [MariaDB公式Dockerイメージ](https://hub.docker.com/_/mariadb)
- [phpMyAdmin公式Dockerイメージ](https://hub.docker.com/_/phpmyadmin)
- [Docker Compose公式ドキュメント](https://docs.docker.com/compose/)
```

**実装のポイント**:

1. **構造**
   - 特徴、前提条件、セットアップ手順の順に記載
   - 各構成パターンの詳細な説明
   - トラブルシューティングの充実

2. **わかりやすさ**
   - コマンド例を豊富に記載
   - 各パターンでの起動コマンドを明示
   - エラーメッセージと対処方法をセットで記載

3. **網羅性**
   - すべての受け入れ基準をカバー
   - 4つの構成パターンすべてを説明
   - データ管理、バージョン更新も含む

---

### 4.6 Phase 6: 動作検証・テスト

#### 4.6.1 テスト計画

**目的**: 要件定義書のすべての受け入れ基準を満たすことを確認する

#### 4.6.2 機能テスト

| テストID | テスト項目 | 手順 | 期待結果 | 対応要件 |
|---------|----------|------|---------|---------|
| FT-01 | Apache + MySQL起動 | `.env`でWP_SERVER=apache, DB_TYPE=mysqlを設定し、`docker-compose up -d`実行 | コンテナが起動し、http://localhost:8080でWordPressにアクセスできる | REQ-WP-001, REQ-DB-001 |
| FT-02 | Apache + MariaDB起動 | `.env`でWP_SERVER=apache, DB_TYPE=mariadbを設定し、`docker-compose up -d`実行 | コンテナが起動し、WordPressにアクセスできる | REQ-DB-001 |
| FT-03 | Nginx + MySQL起動 | `.env`でWP_SERVER=fpmを設定し、`docker-compose --profile nginx up -d`実行 | nginxコンテナ含めて起動し、WordPressにアクセスできる | REQ-WP-002 |
| FT-04 | Nginx + MariaDB起動 | `.env`でWP_SERVER=fpm, DB_TYPE=mariadbを設定し、`docker-compose --profile nginx up -d`実行 | すべてのコンテナが起動し、WordPressにアクセスできる | REQ-WP-002, REQ-DB-001 |
| FT-05 | phpMyAdmin起動 | `PHPMYADMIN_ENABLED=true`を設定し、`docker-compose --profile phpmyadmin up -d`実行 | phpMyAdminコンテナが起動し、http://localhost:8081でアクセスできる | REQ-PMA-001 |
| FT-06 | phpMyAdmin無効 | `PHPMYADMIN_ENABLED=false`を設定し、`docker-compose up -d`実行 | phpMyAdminコンテナが起動しない | REQ-PMA-001 |
| FT-07 | WordPress初期セットアップ | WordPressにアクセスし、セットアップ画面で設定を完了 | セットアップが正常に完了し、管理画面にログインできる | 受け入れ基準5.1 |
| FT-08 | テーマインストール | 管理画面から任意のテーマをインストール | テーマが正常にインストールされる | 受け入れ基準5.1 |
| FT-09 | プラグインインストール | 管理画面から任意のプラグインをインストール | プラグインが正常にインストールされる | 受け入れ基準5.1 |
| FT-10 | メディアアップロード | 64MB以下のファイルをアップロード | ファイルが正常にアップロードされる | NFR-PERF-003 |
| FT-11 | データ永続化確認 | `docker-compose down && docker-compose up -d`を実行 | コンテナ再作成後もWordPressのデータが保持される | REQ-VOL-001, REQ-VOL-002 |
| FT-12 | デバッグログ確認 | エラーを発生させ、`wordpress/wp-content/debug.log`を確認 | エラーがログに記録されている | REQ-DEV-002 |
| FT-13 | phpMyAdmin自動ログイン | phpMyAdminにアクセス | ログイン画面なしで直接データベースにアクセスできる | REQ-PMA-001 |

#### 4.6.3 設定テスト

| テストID | テスト項目 | 手順 | 期待結果 | 対応要件 |
|---------|----------|------|---------|---------|
| CT-01 | PHPバージョン変更 | `.env`でPHP_VERSION=8.1に変更し、コンテナ再起動 | PHP 8.1で動作する | REQ-WP-001 |
| CT-02 | WordPressバージョン変更 | `.env`でWP_VERSION=6.4に変更し、コンテナ再起動 | WordPress 6.4で動作する | REQ-WP-001 |
| CT-03 | データベースバージョン変更 | `.env`でDB_VERSION=8.4に変更し、コンテナ再起動 | MySQL 8.4で動作する | REQ-DB-001 |
| CT-04 | ポート変更 | `.env`でWP_PORT=8000に変更し、コンテナ再起動 | http://localhost:8000でアクセスできる | REQ-WP-003 |
| CT-05 | phpMyAdminポート変更 | `.env`でPHPMYADMIN_PORT=9000に変更し、コンテナ再起動 | http://localhost:9000でphpMyAdminにアクセスできる | REQ-PMA-003 |
| CT-06 | phpMyAdminバージョン変更 | `.env`でPHPMYADMIN_VERSION=5.2に変更し、コンテナ再起動 | phpMyAdmin 5.2で動作する | REQ-PMA-002 |

#### 4.6.4 非機能テスト

| テストID | テスト項目 | 手順 | 期待結果 | 対応要件 |
|---------|----------|------|---------|---------|
| NFT-01 | 初回起動時間 | イメージキャッシュなしで`docker-compose up -d`実行 | 5分以内に起動完了 | NFR-PERF-001 |
| NFT-02 | 2回目以降の起動時間 | イメージキャッシュありで`docker-compose up -d`実行 | 30秒以内に起動完了 | NFR-PERF-001 |
| NFT-03 | 大容量ファイルアップロード | 64MBのファイルをアップロード | 正常にアップロードできる | NFR-PERF-003 |
| NFT-04 | データベースヘルスチェック | `docker-compose ps`でヘルスステータスを確認 | dbコンテナのSTATEが"healthy"になっている | NFR-AVAIL-002 |

#### 4.6.5 セキュリティテスト

| テストID | テスト項目 | 手順 | 期待結果 | 対応要件 |
|---------|----------|------|---------|---------|
| ST-01 | .env除外確認 | `git status`実行 | `.env`ファイルがUntrackedとして表示されない | NFR-SEC-001 |
| ST-02 | wp-content除外確認 | `git status`実行 | `wordpress/wp-content/`（.gitkeep除く）が表示されない | NFR-SEC-001 |

#### 4.6.6 ドキュメントテスト

| テストID | テスト項目 | 手順 | 期待結果 | 対応要件 |
|---------|----------|------|---------|---------|
| DT-01 | README整合性確認 | README.mdの手順に従って操作 | すべての手順が正しく動作する | NFR-MAINT-001 |
| DT-02 | .env.example完全性確認 | `.env.example`の変数を確認 | docker-compose.ymlで使用するすべての変数が記載されている | REQ-ENV-001 |

#### 4.6.7 構成パターンテスト

すべての構成パターンで以下を確認します。

| パターン | WP_SERVER | DB_TYPE | phpMyAdmin | 起動コマンド | 確認項目 |
|---------|-----------|---------|-----------|------------|---------|
| Pattern 1 | apache | mysql | false | `docker-compose up -d` | WordPress動作、データ永続化 |
| Pattern 2 | apache | mariadb | false | `docker-compose up -d` | WordPress動作、データ永続化 |
| Pattern 3 | fpm | mysql | false | `docker-compose --profile nginx up -d` | WordPress動作、Nginx経由アクセス |
| Pattern 4 | fpm | mariadb | true | `docker-compose --profile nginx --profile phpmyadmin up -d` | WordPress動作、phpMyAdminアクセス |

**確認項目**:
1. コンテナが正常に起動する
2. WordPressにアクセスできる
3. WordPress初期セットアップが完了できる
4. データがコンテナ再作成後も保持される
5. phpMyAdmin（有効時）にアクセスできる

#### 4.6.8 テスト実施手順

**事前準備**:
```bash
# 既存のコンテナ・ボリューム削除
docker-compose down -v

# .envファイル削除
rm .env
```

**テスト実行**:
1. `.env.example`から`.env`を作成
2. 各テストケースの設定を`.env`に反映
3. 起動コマンドを実行
4. 期待結果を確認
5. ログを記録
6. 次のテストのため環境をクリーンアップ

**クリーンアップ**:
```bash
docker-compose down -v
```

---

## 5. データ設計

### 5.1 データベーススキーマ

本プロジェクトではWordPress標準のデータベーススキーマを使用します。カスタムテーブルは作成しません。

### 5.2 データ構造とモデル定義

#### 5.2.1 環境変数データ構造

| カテゴリ | 変数名 | データ型 | 必須 | 制約 |
|---------|-------|---------|------|------|
| WordPress | WP_VERSION | string | ○ | - |
| WordPress | PHP_VERSION | string | ○ | 8.1, 8.2, 8.3 |
| WordPress | WP_SERVER | string | ○ | "apache" または "fpm" |
| WordPress | WP_PORT | integer | ○ | 1-65535 |
| Database | DB_TYPE | string | ○ | "mysql" または "mariadb" |
| Database | DB_VERSION | string | ○ | - |
| Database | MYSQL_DATABASE | string | ○ | - |
| Database | MYSQL_USER | string | ○ | - |
| Database | MYSQL_PASSWORD | string | ○ | - |
| Database | MYSQL_ROOT_PASSWORD | string | ○ | - |
| phpMyAdmin | PHPMYADMIN_ENABLED | string | ○ | "true" または "false" |
| phpMyAdmin | PHPMYADMIN_VERSION | string | ○ | - |
| phpMyAdmin | PHPMYADMIN_PORT | integer | ○ | 1-65535 |

### 5.3 データフロー

設計書のセクション3.2「データフロー」を参照してください。

---

## 6. API設計

本プロジェクトでは独自APIを作成しません。WordPress標準のREST APIを使用します。

---

## 7. テスト計画

セクション4.6「Phase 6: 動作検証・テスト」を参照してください。

---

## 8. エラーハンドリング

### 8.1 例外処理の方針

#### 8.1.1 起動時エラー

| エラー種別 | 検出方法 | 対処方法 |
|-----------|---------|---------|
| ポート競合 | docker-composeの起動エラー | `.env`でポート番号を変更 |
| 環境変数未設定 | docker-compose config実行 | `.env`ファイルを作成・設定 |
| イメージダウンロード失敗 | docker-compose up実行時のエラー | ネットワーク接続を確認、再試行 |

#### 8.1.2 実行時エラー

| エラー種別 | 検出方法 | 対処方法 |
|-----------|---------|---------|
| データベース接続失敗 | WordPressのエラー画面 | データベースコンテナのログ確認、ヘルスチェック確認 |
| パーミッションエラー | WordPressのエラー画面 | `chmod -R 755 wordpress/wp-content` |
| Nginx設定エラー | nginxコンテナ起動失敗 | `nginx -t`で構文チェック |

### 8.2 エラーメッセージの設計

#### 8.2.1 ユーザー向けエラーメッセージ

README.mdの「トラブルシューティング」セクションに記載します。

- エラーメッセージ例を明示
- 原因の説明
- 具体的な対処コマンド

#### 8.2.2 ログ出力

| ログ種別 | 出力先 | 確認方法 |
|---------|-------|---------|
| WordPressデバッグログ | `wordpress/wp-content/debug.log` | ホスト側でファイルを開く |
| Dockerコンテナログ | Docker標準出力 | `docker-compose logs` |
| Nginxアクセスログ | `/var/log/nginx/access.log` | `docker-compose exec nginx cat /var/log/nginx/access.log` |
| Nginxエラーログ | `/var/log/nginx/error.log` | `docker-compose exec nginx cat /var/log/nginx/error.log` |

### 8.3 ログ出力の仕様

#### 8.3.1 WordPressデバッグログ

- **出力先**: `wordpress/wp-content/debug.log`
- **フォーマット**: WordPress標準形式
- **出力条件**: `WORDPRESS_DEBUG=1`および`WORDPRESS_DEBUG_LOG=1`の場合

**ログ例**:
```
[17-Feb-2026 10:30:45 UTC] PHP Warning: ...
```

#### 8.3.2 Dockerコンテナログ

- **出力先**: Docker標準出力（JSON形式）
- **確認方法**: `docker-compose logs`

---

## 9. パフォーマンス考慮事項

### 9.1 想定される負荷とスケーラビリティ

本プロジェクトはローカル開発環境用のため、以下を想定します。

- **同時接続数**: 1（開発者のみ）
- **データ量**: 開発用の少量データ
- **トラフィック**: ローカルネットワークのみ

### 9.2 最適化ポイント

#### 9.2.1 イメージサイズ最適化

- **Nginx**: `nginx:alpine`を使用（約25MB、通常版の約1/5）
- **WordPress**: 公式イメージを使用（最適化済み）

#### 9.2.2 起動時間最適化

1. **イメージキャッシュ**
   - 初回起動後、イメージはローカルにキャッシュされる
   - 2回目以降は高速起動

2. **ヘルスチェック**
   - データベースの起動完了を確認してからWordPressを起動
   - 接続エラーによる再起動を防止

3. **depends_on設定**
   - コンテナの起動順序を最適化
   - 不要な待機時間を削減

#### 9.2.3 実行時パフォーマンス

1. **名前付きボリューム使用**
   - データベースデータは名前付きボリュームで管理
   - バインドマウントより高速

2. **Nginx静的ファイル配信**
   - 静的ファイル（画像、CSS、JS）はNginxが直接配信
   - PHP-FPMの負荷を軽減

3. **FastCGIバッファ設定**
   - `fastcgi_buffers 8 16k`
   - `fastcgi_buffer_size 32k`
   - PHP-FPMとNginx間の通信を最適化

### 9.3 モニタリング方法

#### 9.3.1 コンテナ状態監視

```bash
# コンテナの状態確認
docker-compose ps

# リソース使用状況確認
docker stats
```

#### 9.3.2 ログ監視

```bash
# リアルタイムログ監視
docker-compose logs -f

# 特定のサービスのログ
docker-compose logs -f wordpress
```

#### 9.3.3 ヘルスチェック確認

```bash
# ヘルスステータス確認
docker-compose ps
# STATE列が "healthy" になっていることを確認
```

---

## 10. セキュリティ対策

### 10.1 脆弱性対策

#### 10.1.1 コンテナイメージ

| 対策 | 内容 |
|-----|------|
| 公式イメージ使用 | WordPress、MySQL、MariaDB、Nginx、phpMyAdminは公式イメージを使用 |
| バージョン管理 | 環境変数でバージョンを明示的に管理 |
| 定期的な更新 | イメージを定期的に更新して脆弱性に対応 |

#### 10.1.2 Nginx設定

| 対策 | 内容 |
|-----|------|
| .htaccessアクセス拒否 | `location ~ /\.ht { deny all; }` |
| .gitアクセス拒否 | `location ~ /\.git { deny all; }` |
| wp-config.phpアクセス拒否 | `location = /wp-config.php { deny all; }` |
| xmlrpc.phpアクセス拒否 | `location = /xmlrpc.php { deny all; }` |

### 10.2 データ保護の仕組み

#### 10.2.1 機密情報の管理

| 対象 | 保護方法 |
|-----|---------|
| データベース認証情報 | `.env`ファイルで管理、Git除外 |
| WordPress認証情報 | データベース内に暗号化して保存（WordPress標準） |
| phpMyAdmin認証情報 | 環境変数で管理、自動ログイン |

#### 10.2.2 データ永続化

| データ | 保存方法 | バックアップ方法 |
|-------|---------|---------------|
| データベース | 名前付きボリューム（`db_data`） | `mysqldump`コマンド |
| wp-content | バインドマウント | `tar`コマンド |

### 10.3 アクセス制御

#### 10.3.1 ネットワークアクセス制御

| リソース | アクセス元 | 制限 |
|---------|----------|------|
| WordPress | ホスト（localhost） | ポート8080（変更可） |
| データベース | Dockerネットワーク内のみ | ホストに公開しない |
| phpMyAdmin | ホスト（localhost） | ポート8081（変更可） |

#### 10.3.2 ローカル環境限定

- **対象**: 本設定はローカル開発環境専用
- **制限**: 本番環境での使用を禁止
- **理由**: デバッグモード有効、簡易パスワード使用

---

## 11. 保守性

### 11.1 コード品質

#### 11.1.1 YAMLフォーマット

- インデント: スペース2つ
- コメント: 各セクションに説明を記載
- 構造: サービス、ネットワーク、ボリュームの順

#### 11.1.2 環境変数命名規則

- 大文字とアンダースコアを使用（例: `WP_VERSION`）
- 接頭辞でカテゴリを識別（例: `WP_`, `DB_`, `MYSQL_`, `PHPMYADMIN_`）

### 11.2 ドキュメント保守

#### 11.2.1 README.md

- セクション構成を明確化
- コマンド例を豊富に記載
- トラブルシューティングを充実

#### 11.2.2 .env.example

- すべての環境変数を記載
- 各変数に説明コメントを付ける
- デフォルト値を明示

#### 11.2.3 docker-compose.yml

- 各サービスにコメントを記載
- 設定項目の目的を説明
- 参照する環境変数を明示

### 11.3 バージョン管理

#### 11.3.1 Git管理対象

- `.gitignore`
- `.env.example`
- `docker-compose.yml`
- `nginx/nginx.conf`
- `README.md`
- `wordpress/wp-content/.gitkeep`

#### 11.3.2 Git管理除外

- `.env`（機密情報）
- `wordpress/wp-content/`（生成ファイル、.gitkeep除く）
- `*.log`（ログファイル）
- `*.sql`（バックアップファイル）

---

## 12. 実装チェックリスト

### 12.1 Phase 1: 基本ファイル作成

- [ ] `.gitignore`作成
- [ ] `.env.example`作成
- [ ] `wordpress/wp-content/`ディレクトリ作成
- [ ] `wordpress/wp-content/.gitkeep`作成

### 12.2 Phase 2: docker-compose.yml実装

- [ ] `docker-compose.yml`作成
- [ ] `wordpress`サービス定義
- [ ] `db`サービス定義
- [ ] `nginx`サービス定義（profiles使用）
- [ ] `phpmyadmin`サービス定義（profiles使用）
- [ ] ネットワーク定義
- [ ] ボリューム定義
- [ ] 環境変数参照設定
- [ ] ヘルスチェック設定
- [ ] YAML構文チェック（`docker-compose config`）

### 12.3 Phase 3: Nginx設定ファイル作成

- [ ] `nginx/`ディレクトリ作成
- [ ] `nginx/nginx.conf`作成
- [ ] FastCGI設定
- [ ] WordPress用ルーティング設定
- [ ] 静的ファイルキャッシュ設定
- [ ] セキュリティ設定（.htaccess、.git等のアクセス拒否）
- [ ] Nginx設定構文チェック（`nginx -t`）

### 12.4 Phase 4: phpMyAdmin統合

- [ ] `phpmyadmin`サービスがdocker-compose.ymlに定義されていること
- [ ] `profiles: ["phpmyadmin"]`設定
- [ ] 環境変数設定（PMA_HOST、PMA_USER、PMA_PASSWORD）

### 12.5 Phase 5: README.md作成

- [ ] プロジェクト概要記載
- [ ] 前提条件記載
- [ ] セットアップ手順記載
- [ ] 4つの構成パターン説明
- [ ] 基本操作コマンド記載
- [ ] データ管理方法記載
- [ ] トラブルシューティング記載
- [ ] ファイル構成記載

### 12.6 Phase 6: 動作検証・テスト

- [ ] FT-01: Apache + MySQL起動テスト
- [ ] FT-02: Apache + MariaDB起動テスト
- [ ] FT-03: Nginx + MySQL起動テスト
- [ ] FT-04: Nginx + MariaDB起動テスト
- [ ] FT-05: phpMyAdmin起動テスト
- [ ] FT-06: phpMyAdmin無効テスト
- [ ] FT-07: WordPress初期セットアップテスト
- [ ] FT-08: テーマインストールテスト
- [ ] FT-09: プラグインインストールテスト
- [ ] FT-10: メディアアップロードテスト
- [ ] FT-11: データ永続化確認テスト
- [ ] FT-12: デバッグログ確認テスト
- [ ] FT-13: phpMyAdmin自動ログインテスト
- [ ] CT-01: PHPバージョン変更テスト
- [ ] CT-02: WordPressバージョン変更テスト
- [ ] CT-03: データベースバージョン変更テスト
- [ ] CT-04: ポート変更テスト
- [ ] CT-05: phpMyAdminポート変更テスト
- [ ] CT-06: phpMyAdminバージョン変更テスト
- [ ] NFT-01: 初回起動時間テスト
- [ ] NFT-02: 2回目以降の起動時間テスト
- [ ] NFT-03: 大容量ファイルアップロードテスト
- [ ] NFT-04: データベースヘルスチェックテスト
- [ ] ST-01: .env除外確認テスト
- [ ] ST-02: wp-content除外確認テスト
- [ ] DT-01: README整合性確認テスト
- [ ] DT-02: .env.example完全性確認テスト

---

## 13. リスクと対策

### 13.1 実装リスク

| リスクID | リスク内容 | 影響度 | 発生確率 | 対策 |
|---------|----------|--------|---------|------|
| R-01 | ポート競合によるコンテナ起動失敗 | 中 | 高 | README.mdにトラブルシューティングを記載、環境変数でポート変更可能 |
| R-02 | 環境変数の設定ミス | 中 | 中 | .env.exampleに詳細なコメントを記載、デフォルト値を設定 |
| R-03 | パーミッションエラー | 低 | 中 | README.mdに対処方法を記載 |
| R-04 | データベース起動遅延 | 低 | 低 | ヘルスチェック設定で対処 |
| R-05 | Nginx設定エラー | 中 | 低 | 詳細なコメントと構文チェックコマンドを提供 |

### 13.2 運用リスク

| リスクID | リスク内容 | 影響度 | 発生確率 | 対策 |
|---------|----------|--------|---------|------|
| R-06 | データの誤削除（docker-compose down -v） | 高 | 中 | README.mdに警告を記載、バックアップ方法を提供 |
| R-07 | .envファイルのGitコミット | 高 | 低 | .gitignoreで除外、.env.exampleに注意書き |
| R-08 | 本番環境での誤使用 | 高 | 低 | README.mdとdocker-compose.ymlに「ローカル環境専用」と明記 |

---

## 14. 今後の拡張可能性

### 14.1 追加機能候補

実装完了後、以下の機能追加を検討可能です。

| 機能 | 優先度 | 実装難易度 | 説明 |
|-----|-------|----------|------|
| WP-CLI統合 | 中 | 低 | コマンドラインでのWordPress操作 |
| Xdebug統合 | 中 | 中 | PHPデバッグ機能 |
| Mailhog統合 | 低 | 低 | メール送信テスト |
| Redis統合 | 低 | 中 | オブジェクトキャッシュ |
| HTTPS対応 | 低 | 中 | mkcert等を使用したローカルSSL |
| 起動スクリプト | 中 | 低 | 環境変数を読み取り自動的に適切なprofileで起動 |
| 環境変数検証スクリプト | 中 | 低 | 起動前に環境変数を検証 |

### 14.2 改善ポイント

| 項目 | 内容 |
|-----|------|
| ドキュメント | 動画チュートリアルの作成 |
| 自動化 | Makefileの作成（起動、停止、バックアップ等のコマンドを簡略化） |
| テスト | 自動テストスクリプトの作成 |

---

## 15. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|----------|------|---------|--------|
| 1.0 | 2026-02-17 | 初版作成 | Claude Code |

---

## 16. 承認

| 役割 | 氏名 | 承認日 | 署名 |
|-----|------|--------|------|
| 実装計画者 | Claude Code | 2026-02-17 | - |
| レビュアー | - | - | - |
| 承認者 | - | - | - |

---

## 17. 付録

### 17.1 参考コマンド集

#### 17.1.1 Docker Compose基本コマンド

```bash
# 起動
docker-compose up -d

# 停止
docker-compose stop

# 再起動
docker-compose restart

# 削除（ボリューム保持）
docker-compose down

# 削除（ボリューム削除）
docker-compose down -v

# ログ確認
docker-compose logs
docker-compose logs -f

# コンテナ状態確認
docker-compose ps

# 設定確認
docker-compose config
```

#### 17.1.2 データベース操作コマンド

```bash
# バックアップ
docker-compose exec db mysqldump -u wordpress -pwordpress wordpress > backup.sql

# リストア
docker-compose exec -T db mysql -u wordpress -pwordpress wordpress < backup.sql

# MySQLコンソール接続
docker-compose exec db mysql -u wordpress -pwordpress wordpress
```

#### 17.1.3 WordPress操作コマンド

```bash
# コンテナ内でbashを起動
docker-compose exec wordpress bash

# wp-contentのパーミッション修正
chmod -R 755 wordpress/wp-content

# デバッグログ確認
tail -f wordpress/wp-content/debug.log
```

#### 17.1.4 Nginx操作コマンド

```bash
# 設定ファイル構文チェック
docker-compose run --rm nginx nginx -t

# 設定リロード
docker-compose exec nginx nginx -s reload

# アクセスログ確認
docker-compose exec nginx tail -f /var/log/nginx/access.log

# エラーログ確認
docker-compose exec nginx tail -f /var/log/nginx/error.log
```

### 17.2 環境変数クイックリファレンス

#### 17.2.1 Apache + MySQL構成

```bash
WP_SERVER=apache
DB_TYPE=mysql
DB_VERSION=8.0
PHPMYADMIN_ENABLED=false
```

#### 17.2.2 Nginx + MariaDB + phpMyAdmin構成

```bash
WP_SERVER=fpm
DB_TYPE=mariadb
DB_VERSION=11.4
PHPMYADMIN_ENABLED=true
```

起動コマンド:
```bash
docker-compose --profile nginx --profile phpmyadmin up -d
```

### 17.3 トラブルシューティング早見表

| 症状 | 原因 | 対処 |
|-----|------|------|
| ポート8080でアクセスできない | ポート競合 | `.env`でポート変更 |
| データベース接続エラー | DB未起動 | `docker-compose logs db`確認 |
| パーミッションエラー | ファイル権限 | `chmod -R 755 wordpress/wp-content` |
| Nginx起動失敗 | 設定エラー | `nginx -t`で構文チェック |
| phpMyAdmin起動しない | profile未指定 | `--profile phpmyadmin`を指定 |

---

**この実装計画に対するフィードバックをお願いいたします。**
