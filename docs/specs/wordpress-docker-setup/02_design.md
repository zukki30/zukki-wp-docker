# 設計書: WordPress Docker環境構築

## ドキュメント情報
- **spec名**: wordpress-docker-setup
- **作成日**: 2026-02-17
- **最終更新日**: 2026-02-17
- **ステータス**: レビュー待ち
- **バージョン**: 1.0

---

## 1. システム概要

### 1.1 設計の全体像

本設計は、Docker Composeを用いたWordPress開発環境を構築するものです。以下の特徴を持ちます。

- **柔軟な構成切り替え**: 環境変数のみで、Webサーバー（Apache/Nginx）、データベース（MySQL/MariaDB）、PHPバージョン、phpMyAdminの有効/無効を切り替え可能
- **データ永続化**: コンテナ再作成後もデータを保持
- **開発最適化**: デバッグモード有効化、エラーログ出力、自動起動設定
- **シンプルな操作**: `.env`ファイルのコピーと`docker-compose up`のみで起動

### 1.2 主要コンポーネント

システムは以下の4つのコンテナで構成されます。

1. **WordPressコンテナ** (`wordpress`)
   - Apache使用時: `wordpress:php${PHP_VERSION}-apache`
   - Nginx使用時: `wordpress:php${PHP_VERSION}-fpm`

2. **データベースコンテナ** (`db`)
   - MySQL使用時: `mysql:${DB_VERSION}`
   - MariaDB使用時: `mariadb:${DB_VERSION}`

3. **Nginxコンテナ** (`nginx`) ※Nginx使用時のみ
   - イメージ: `nginx:alpine`
   - WordPressコンテナ（FPM）とFastCGI通信

4. **phpMyAdminコンテナ** (`phpmyadmin`) ※有効時のみ
   - イメージ: `phpmyadmin:${PHPMYADMIN_VERSION}`
   - データベース管理用WebUI

### 1.3 アーキテクチャ図

```
┌─────────────────────────────────────────────────────────────┐
│                        ホストマシン                          │
│                                                               │
│  ┌────────────────────────────────────────────────────────┐ │
│  │                   Docker Network                        │ │
│  │                                                          │ │
│  │  ┌─────────────────┐                                    │ │
│  │  │   WordPress     │  ◄── Apache使用時                 │ │
│  │  │   (apache)      │      ポート: 8080                 │ │
│  │  └─────────────────┘                                    │ │
│  │          OR                                              │ │
│  │  ┌─────────────────┐    ┌─────────────────┐            │ │
│  │  │   WordPress     │◄───┤     Nginx       │            │ │
│  │  │     (fpm)       │    │    (alpine)     │            │ │
│  │  │                 │    │  ポート: 8080   │◄── Nginx  │ │
│  │  └─────────────────┘    └─────────────────┘   使用時   │ │
│  │          │                                               │ │
│  │          │ FastCGI                                       │ │
│  │          ▼                                               │ │
│  │  ┌─────────────────┐                                    │ │
│  │  │   Database      │  ◄── MySQL または MariaDB         │ │
│  │  │   (mysql/maria) │      内部ポート: 3306             │ │
│  │  └─────────────────┘                                    │ │
│  │          ▲                                               │ │
│  │          │                                               │ │
│  │          │ 接続                                          │ │
│  │  ┌─────────────────┐                                    │ │
│  │  │  phpMyAdmin     │  ◄── 有効時のみ起動               │ │
│  │  │   (optional)    │      ポート: 8081                 │ │
│  │  └─────────────────┘                                    │ │
│  │                                                          │ │
│  └────────────────────────────────────────────────────────┘ │
│                                                               │
│  ボリューム:                                                 │
│  - ./wordpress/wp-content ◄─► /var/www/html/wp-content      │
│  - db_data (名前付きボリューム) ◄─► /var/lib/mysql          │
│  - ./nginx/nginx.conf ◄─► /etc/nginx/nginx.conf (Nginx時) │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. アーキテクチャ設計

### 2.1 システム構成パターン

本設計では、環境変数の組み合わせにより以下の構成パターンをサポートします。

#### 2.1.1 構成パターン一覧

| パターン | WP_SERVER | DB_TYPE | Nginxコンテナ | 主な用途 |
|---------|-----------|---------|--------------|---------|
| Pattern 1 | apache | mysql | 不要 | 標準的なLAMP環境 |
| Pattern 2 | apache | mariadb | 不要 | MariaDB使用環境 |
| Pattern 3 | fpm | mysql | 必要 | Nginx + MySQL環境 |
| Pattern 4 | fpm | mariadb | 必要 | Nginx + MariaDB環境 |

※ 各パターンでphpMyAdminの有効/無効を選択可能

#### 2.1.2 コンテナ起動制御

- **Apache使用時**: `wordpress`コンテナのみ起動（ポート8080を公開）
- **Nginx使用時**: `wordpress`（FPM）と`nginx`コンテナを起動（nginxがポート8080を公開）
- **phpMyAdmin**: `PHPMYADMIN_ENABLED=true`の場合のみ起動

### 2.2 コンテナ間通信設計

#### 2.2.1 ネットワーク構成

- **ネットワーク名**: デフォルト（Docker Composeが自動生成）
- **通信方式**: Dockerブリッジネットワーク
- **サービス名解決**: Docker内蔵DNSによる自動解決

#### 2.2.2 通信経路

1. **ホスト → WordPress**
   - Apache使用時: `http://localhost:8080` → `wordpress:80`
   - Nginx使用時: `http://localhost:8080` → `nginx:80` → `wordpress:9000` (FastCGI)

2. **WordPress → Database**
   - ホスト名: `db`（Docker内蔵DNSで解決）
   - ポート: `3306`
   - プロトコル: MySQL/MariaDBプロトコル

3. **phpMyAdmin → Database**
   - ホスト名: `db`
   - ポート: `3306`
   - 自動ログイン設定を使用

4. **Nginx → WordPress (FPM)**
   - ホスト名: `wordpress`
   - ポート: `9000`
   - プロトコル: FastCGI

### 2.3 ポート設計

#### 2.3.1 ポートマッピング

| コンテナ | 内部ポート | ホストポート | 環境変数 | 公開条件 |
|---------|----------|------------|---------|---------|
| wordpress (apache) | 80 | 8080 | WP_PORT | WP_SERVER=apache時 |
| nginx | 80 | 8080 | WP_PORT | WP_SERVER=fpm時 |
| db | 3306 | - | - | 公開しない（内部のみ） |
| phpmyadmin | 80 | 8081 | PHPMYADMIN_PORT | PHPMYADMIN_ENABLED=true時 |

#### 2.3.2 ポート競合回避設計

- **WordPress**: デフォルト8080を使用（環境変数`WP_PORT`で変更可能）
- **phpMyAdmin**: デフォルト8081を使用（環境変数`PHPMYADMIN_PORT`で変更可能）
- **Database**: ホストに公開しない（セキュリティとポート競合回避）

---

## 3. データ設計

### 3.1 データベース設計

#### 3.1.1 データベース仕様

| 項目 | 設定値 | 環境変数 |
|-----|-------|---------|
| データベース名 | wordpress | MYSQL_DATABASE |
| ユーザー名 | wordpress | MYSQL_USER |
| パスワード | wordpress | MYSQL_PASSWORD |
| rootパスワード | rootpassword | MYSQL_ROOT_PASSWORD |
| 文字コード | utf8mb4 | - |
| 照合順序 | utf8mb4_unicode_ci | - |

#### 3.1.2 データベース種類別設定

**MySQL設定**
```yaml
environment:
  MYSQL_DATABASE: ${MYSQL_DATABASE}
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
```

**MariaDB設定**
```yaml
environment:
  MYSQL_DATABASE: ${MYSQL_DATABASE}  # MariaDBも同じ変数名を使用
  MYSQL_USER: ${MYSQL_USER}
  MYSQL_PASSWORD: ${MYSQL_PASSWORD}
  MYSQL_ROOT_PASSWORD: ${MYSQL_ROOT_PASSWORD}
```

※ MySQLとMariaDBは環境変数名に互換性があるため、同じ変数を使用可能

### 3.2 データフロー

#### 3.2.1 WordPress初期セットアップ時

```
ユーザー
  │
  │ (1) http://localhost:8080にアクセス
  ▼
Apache/Nginx
  │
  │ (2) PHPリクエスト転送
  ▼
WordPress (PHP-FPM/mod_php)
  │
  │ (3) データベース接続（ホスト名: db）
  ▼
MySQL/MariaDB
  │
  │ (4) wordpress データベース作成/接続確認
  ▼
WordPress セットアップ画面表示
```

#### 3.2.2 メディアアップロード時

```
ユーザー
  │
  │ (1) 管理画面からファイルアップロード
  ▼
WordPress
  │
  │ (2) ファイルを /var/www/html/wp-content/uploads/ に保存
  │     （ホストの ./wordpress/wp-content/uploads/ にマウント）
  ▼
ホストファイルシステム
  │
  │ (3) メタデータをデータベースに保存
  ▼
MySQL/MariaDB
```

### 3.3 ボリューム設計

#### 3.3.1 ボリューム一覧

| ボリューム名/パス | タイプ | マウント先 | 目的 |
|----------------|-------|----------|------|
| `./wordpress/wp-content` | バインドマウント | `/var/www/html/wp-content` | テーマ、プラグイン、アップロードファイルの永続化 |
| `db_data` | 名前付きボリューム | `/var/lib/mysql` | データベースデータの永続化 |
| `./nginx/nginx.conf` | バインドマウント | `/etc/nginx/nginx.conf` | Nginx設定（fpm時のみ） |

#### 3.3.2 ボリューム選定理由

**wp-content: バインドマウント**
- 理由: ホスト側でファイル編集が必要（テーマ/プラグイン開発）
- メリット: リアルタイム反映、エディタでの直接編集
- デメリット: パーミッション問題の可能性（設計で対応）

**db_data: 名前付きボリューム**
- 理由: データベースファイルはホスト側で直接操作不要
- メリット: パフォーマンス向上、プラットフォーム間の互換性
- デメリット: バックアップに`docker volume`コマンドが必要

#### 3.3.3 ディレクトリ構成

```
zukki-wp-docker/
├── docker-compose.yml
├── .env
├── .env.example
├── .gitignore
├── README.md
├── wordpress/
│   └── wp-content/          # WordPressコンテナからマウント
│       ├── themes/          # テーマ格納
│       ├── plugins/         # プラグイン格納
│       ├── uploads/         # アップロードファイル
│       └── debug.log        # デバッグログ（自動生成）
└── nginx/
    └── nginx.conf           # Nginx設定ファイル（fpm時のみ使用）
```

#### 3.3.4 パーミッション設計

- **wp-content**: `www-data:www-data`（UID:GID = 33:33）
- **対策**: 初回起動時にWordPressコンテナが自動的に適切なパーミッションを設定
- **開発時**: ホスト側でもファイル編集可能なようにパーミッションを調整

---

## 4. 環境変数設計

### 4.1 環境変数一覧

#### 4.1.1 WordPress関連

| 変数名 | 説明 | デフォルト値 | 必須 | 値の例 |
|-------|------|------------|------|-------|
| `WP_VERSION` | WordPressバージョン | latest | ○ | latest, 6.4, 6.3 |
| `PHP_VERSION` | PHPバージョン | 8.2 | ○ | 8.1, 8.2, 8.3 |
| `WP_SERVER` | Webサーバー種類 | apache | ○ | apache, fpm |
| `WP_PORT` | ホスト側公開ポート | 8080 | ○ | 8080, 8000, 3000 |
| `WORDPRESS_DB_HOST` | データベースホスト | db:3306 | ○ | db:3306（固定） |
| `WORDPRESS_DB_NAME` | データベース名 | wordpress | ○ | - |
| `WORDPRESS_DB_USER` | DBユーザー名 | wordpress | ○ | - |
| `WORDPRESS_DB_PASSWORD` | DBパスワード | wordpress | ○ | - |
| `WORDPRESS_DEBUG` | デバッグモード | 1 | ○ | 1（有効）, 0（無効） |
| `WORDPRESS_DEBUG_LOG` | デバッグログ出力 | 1 | ○ | 1（有効）, 0（無効） |
| `WORDPRESS_DEBUG_DISPLAY` | デバッグ画面表示 | 1 | ○ | 1（有効）, 0（無効） |

#### 4.1.2 データベース関連

| 変数名 | 説明 | デフォルト値 | 必須 | 値の例 |
|-------|------|------------|------|-------|
| `DB_TYPE` | データベース種類 | mysql | ○ | mysql, mariadb |
| `DB_VERSION` | データベースバージョン | 8.0 | ○ | MySQL: 8.0, 8.4 / MariaDB: 10.11, 11.4 |
| `MYSQL_DATABASE` | データベース名 | wordpress | ○ | - |
| `MYSQL_USER` | ユーザー名 | wordpress | ○ | - |
| `MYSQL_PASSWORD` | パスワード | wordpress | ○ | - |
| `MYSQL_ROOT_PASSWORD` | rootパスワード | rootpassword | ○ | - |

#### 4.1.3 phpMyAdmin関連

| 変数名 | 説明 | デフォルト値 | 必須 | 値の例 |
|-------|------|------------|------|-------|
| `PHPMYADMIN_ENABLED` | phpMyAdmin有効化 | false | ○ | true, false |
| `PHPMYADMIN_VERSION` | phpMyAdminバージョン | latest | ○ | latest, 5.2, 5.1 |
| `PHPMYADMIN_PORT` | ホスト側公開ポート | 8081 | ○ | 8081, 8082, 9000 |
| `PMA_HOST` | データベースホスト | db | ○ | db（固定） |
| `PMA_USER` | 自動ログインユーザー | wordpress | × | - |
| `PMA_PASSWORD` | 自動ログインパスワード | wordpress | × | - |

### 4.2 .env.example ファイル設計

```bash
# ============================================================
# WordPress Docker環境 設定ファイル
# ============================================================
# このファイルをコピーして .env ファイルを作成してください
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
# - false: phpMyAdminコンテナは起動しません
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

### 4.3 環境変数の参照方法

#### 4.3.1 docker-compose.ymlでの参照

```yaml
# イメージタグの動的構成
image: ${DB_TYPE}:${DB_VERSION}

# 環境変数の受け渡し
environment:
  WORDPRESS_DB_HOST: db:3306
  WORDPRESS_DB_NAME: ${MYSQL_DATABASE}
  WORDPRESS_DB_USER: ${MYSQL_USER}
  WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD}

# ポートマッピングの動的構成
ports:
  - "${WP_PORT}:80"
```

#### 4.3.2 環境変数の検証

以下の条件で環境変数を検証する必要があります（実装計画フェーズで詳細化）。

- `WP_SERVER`が`apache`または`fpm`であること
- `DB_TYPE`が`mysql`または`mariadb`であること
- `PHPMYADMIN_ENABLED`が`true`または`false`であること
- ポート番号が1-65535の範囲内であること

---

## 5. Nginx設定設計（WP_SERVER=fpm時）

### 5.1 nginx.conf構造

#### 5.1.1 基本構造

```nginx
events {
    worker_connections 1024;
}

http {
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

    # アップロードサイズ制限
    client_max_body_size 64M;

    # サーバー設定
    server {
        listen 80;
        server_name localhost;

        root /var/www/html;
        index index.php;

        # WordPress設定
        location / {
            try_files $uri $uri/ /index.php?$args;
        }

        # PHP-FPM設定
        location ~ \.php$ {
            fastcgi_split_path_info ^(.+\.php)(/.+)$;
            fastcgi_pass wordpress:9000;
            fastcgi_index index.php;
            include fastcgi_params;
            fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
            fastcgi_param PATH_INFO $fastcgi_path_info;
        }

        # 静的ファイルのキャッシュ設定
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg)$ {
            expires max;
            log_not_found off;
        }

        # アクセス禁止ファイル
        location ~ /\.ht {
            deny all;
        }

        location ~ /\.git {
            deny all;
        }
    }
}
```

### 5.2 FastCGI設定詳細

#### 5.2.1 FastCGI通信設定

| パラメータ | 値 | 説明 |
|-----------|-----|------|
| `fastcgi_pass` | `wordpress:9000` | PHP-FPMコンテナのホスト名とポート |
| `fastcgi_index` | `index.php` | デフォルトインデックスファイル |
| `SCRIPT_FILENAME` | `$document_root$fastcgi_script_name` | 実行するPHPファイルのフルパス |
| `PATH_INFO` | `$fastcgi_path_info` | パス情報（WordPress用） |

#### 5.2.2 パフォーマンス設定

| パラメータ | 値 | 理由 |
|-----------|-----|------|
| `client_max_body_size` | `64M` | メディアアップロード上限（要件NFR-PERF-003） |
| `fastcgi_buffers` | `8 16k` | FastCGIバッファサイズ |
| `fastcgi_buffer_size` | `32k` | FastCGI初期バッファサイズ |

### 5.3 Nginxコンテナボリューム設計

```yaml
nginx:
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro  # 読み取り専用
    - ./wordpress/wp-content:/var/www/html/wp-content:ro  # 静的ファイル配信用
```

**設計意図**:
- `nginx.conf`は読み取り専用（`:ro`）で安全性確保
- `wp-content`を読み取り専用でマウントし、静的ファイル（画像、CSS、JS）を直接配信
- PHPファイルはFastCGI経由でWordPressコンテナに処理を委譲

---

## 6. docker-compose.yml構造設計

### 6.1 サービス定義設計

#### 6.1.1 サービス一覧

| サービス名 | 起動条件 | 依存関係 | 役割 |
|----------|---------|---------|------|
| `wordpress` | 常時 | `db` | WordPress本体 |
| `db` | 常時 | なし | データベース |
| `nginx` | `WP_SERVER=fpm`時 | `wordpress` | リバースプロキシ・Webサーバー |
| `phpmyadmin` | `PHPMYADMIN_ENABLED=true`時 | `db` | データベース管理UI |

#### 6.1.2 profiles機能の活用

Docker Composeの`profiles`機能を使用して、条件付きでコンテナを起動します。

**設計方針**:
- **Nginxコンテナ**: `profiles: ["nginx"]`を設定
- **phpMyAdminコンテナ**: `profiles: ["phpmyadmin"]`を設定
- **起動方法**: 環境変数に基づいて`--profile`オプションを指定

**起動コマンド例**:
```bash
# Apache + phpMyAdmin無効
docker-compose up -d

# Nginx + phpMyAdmin無効
docker-compose --profile nginx up -d

# Apache + phpMyAdmin有効
docker-compose --profile phpmyadmin up -d

# Nginx + phpMyAdmin有効
docker-compose --profile nginx --profile phpmyadmin up -d
```

※ 実装フェーズでは、環境変数を読み取って自動的に適切なprofileを指定するラッパースクリプトを検討

### 6.2 WordPressサービス設計

#### 6.2.1 基本設定（Apache使用時）

```yaml
wordpress:
  image: wordpress:php${PHP_VERSION:-8.2}-apache
  container_name: wordpress-apache
  restart: unless-stopped
  ports:
    - "${WP_PORT:-8080}:80"
  environment:
    WORDPRESS_DB_HOST: db:3306
    WORDPRESS_DB_NAME: ${MYSQL_DATABASE:-wordpress}
    WORDPRESS_DB_USER: ${MYSQL_USER:-wordpress}
    WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:-wordpress}
    WORDPRESS_DEBUG: 1
    WORDPRESS_DEBUG_LOG: 1
    WORDPRESS_DEBUG_DISPLAY: 1
  volumes:
    - ./wordpress/wp-content:/var/www/html/wp-content
  depends_on:
    - db
  networks:
    - wordpress-network
```

#### 6.2.2 FPM設定（Nginx使用時）

```yaml
wordpress:
  image: wordpress:php${PHP_VERSION:-8.2}-fpm
  container_name: wordpress-fpm
  restart: unless-stopped
  # ポートマッピングなし（nginxコンテナが公開）
  environment:
    WORDPRESS_DB_HOST: db:3306
    WORDPRESS_DB_NAME: ${MYSQL_DATABASE:-wordpress}
    WORDPRESS_DB_USER: ${MYSQL_USER:-wordpress}
    WORDPRESS_DB_PASSWORD: ${MYSQL_PASSWORD:-wordpress}
    WORDPRESS_DEBUG: 1
    WORDPRESS_DEBUG_LOG: 1
    WORDPRESS_DEBUG_DISPLAY: 1
  volumes:
    - ./wordpress/wp-content:/var/www/html/wp-content
  depends_on:
    - db
  networks:
    - wordpress-network
```

**設計上の課題と解決策**:
- **課題**: Apache/FPMで異なるイメージとポート設定が必要
- **解決策**: 環境変数`WP_SERVER`を参照し、Composeファイル内で条件分岐する方法を検討
  - 方法1: `docker-compose.override.yml`を使用
  - 方法2: テンプレートエンジンで動的生成
  - **推奨**: 方法1（Composeの標準機能を活用）

### 6.3 データベースサービス設計

#### 6.3.1 基本設定

```yaml
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
    - db_data:/var/lib/mysql
  networks:
    - wordpress-network
  healthcheck:
    test: ["CMD", "mysqladmin", "ping", "-h", "localhost", "-u", "root", "-p${MYSQL_ROOT_PASSWORD:-rootpassword}"]
    interval: 10s
    timeout: 5s
    retries: 5
```

#### 6.3.2 ヘルスチェック設計

**目的**: データベースが完全に起動してから、WordPressが接続を試みるようにする

**設定内容**:
- **テストコマンド**: `mysqladmin ping`（MySQL/MariaDB共通）
- **間隔**: 10秒ごとにチェック
- **タイムアウト**: 5秒
- **リトライ**: 5回まで

**依存関係の強化**:
```yaml
wordpress:
  depends_on:
    db:
      condition: service_healthy  # ヘルスチェック通過を待つ
```

### 6.4 Nginxサービス設計

```yaml
nginx:
  image: nginx:alpine
  container_name: wordpress-nginx
  restart: unless-stopped
  profiles:
    - nginx
  ports:
    - "${WP_PORT:-8080}:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    - ./wordpress/wp-content:/var/www/html/wp-content:ro
  depends_on:
    - wordpress
  networks:
    - wordpress-network
```

**設計のポイント**:
- **profiles**: `nginx`プロファイルに設定（Apache使用時は起動しない）
- **ポート**: WordPressの代わりに8080ポートを公開
- **ボリューム**: 設定ファイルとwp-content（静的ファイル配信用）をマウント

### 6.5 phpMyAdminサービス設計

```yaml
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
```

**設計のポイント**:
- **profiles**: `phpmyadmin`プロファイルに設定（無効時は起動しない）
- **自動ログイン**: `PMA_USER`と`PMA_PASSWORD`を設定し、ログイン画面をスキップ
- **データベース接続**: `PMA_HOST=db`でデータベースコンテナに自動接続

### 6.6 ネットワーク設計

```yaml
networks:
  wordpress-network:
    driver: bridge
```

**設計のポイント**:
- **ネットワーク名**: `wordpress-network`（明示的に定義）
- **ドライバ**: `bridge`（デフォルト、コンテナ間通信に最適）
- **DNS**: Docker内蔵DNSによるサービス名解決（`db`, `wordpress`, `nginx`等）

### 6.7 ボリューム定義

```yaml
volumes:
  db_data:
    driver: local
```

**設計のポイント**:
- **名前付きボリューム**: `db_data`（データベースデータ永続化）
- **ドライバ**: `local`（ローカルストレージ）
- **削除保護**: `docker-compose down`では削除されない（`-v`オプション時のみ削除）

---

## 7. セキュリティ設計

### 7.1 機密情報保護

#### 7.1.1 .env管理

| ファイル | 用途 | Git管理 |
|---------|------|--------|
| `.env` | 実際の設定値（機密情報含む） | **管理しない** |
| `.env.example` | サンプル設定（機密情報なし） | 管理する |

#### 7.1.2 .gitignore設計

```gitignore
# 環境設定ファイル
.env

# WordPress データ
wordpress/wp-content/

# データベースバックアップ
*.sql
*.sql.gz

# ログファイル
*.log

# macOS
.DS_Store

# IDE
.vscode/
.idea/
```

### 7.2 ローカル環境限定設計

本設計はローカル開発環境専用であり、以下の設定は本番環境では使用しないでください。

- **デバッグモード有効**: `WORDPRESS_DEBUG=1`
- **簡易パスワード**: デフォルト値を使用
- **ポート公開**: ホストに直接ポート公開

### 7.3 アクセス制御

| リソース | アクセス制限 |
|---------|------------|
| WordPress管理画面 | なし（ローカル環境のため） |
| phpMyAdmin | なし（ローカル環境のため） |
| データベースポート | ホストに公開しない |

---

## 8. パフォーマンス設計

### 8.1 メモリ設定

#### 8.1.1 PHPメモリ制限

WordPress公式イメージのデフォルト設定を使用します。

- **メモリ制限**: 256MB（WordPress推奨値）
- **アップロードサイズ**: 64MB（nginx.confで設定）

#### 8.1.2 追加設定（必要に応じて）

カスタムPHP設定が必要な場合、以下の方法で設定可能です。

**方法1: カスタムphp.ini**
```yaml
wordpress:
  volumes:
    - ./php-custom.ini:/usr/local/etc/php/conf.d/custom.ini
```

**php-custom.ini例**:
```ini
memory_limit = 512M
upload_max_filesize = 128M
post_max_size = 128M
max_execution_time = 300
```

### 8.2 起動時間最適化

#### 8.2.1 イメージキャッシュ戦略

- **初回起動**: イメージダウンロードに時間がかかる（目標5分以内）
- **2回目以降**: ローカルキャッシュを使用（目標30秒以内）

#### 8.2.2 depends_onとhealthcheck

```yaml
wordpress:
  depends_on:
    db:
      condition: service_healthy
```

**効果**: データベース起動完了を待ってからWordPressを起動し、接続エラーを防止

---

## 9. エラーハンドリング設計

### 9.1 起動失敗時の対応

#### 9.1.1 エラーパターンと対処

| エラー | 原因 | 対処方法 |
|-------|------|---------|
| ポート競合 | 既存サービスが同じポートを使用 | `.env`でポート番号を変更 |
| データベース接続失敗 | DBコンテナ未起動 | `docker-compose logs db`でログ確認 |
| パーミッションエラー | `wp-content`の権限問題 | `chmod -R 755 wordpress/wp-content` |
| Nginx起動失敗 | `nginx.conf`の文法エラー | `docker-compose logs nginx`でログ確認 |

#### 9.1.2 ログ確認コマンド

```bash
# すべてのコンテナログ
docker-compose logs

# 特定のサービスログ
docker-compose logs wordpress
docker-compose logs db
docker-compose logs nginx

# リアルタイムログ監視
docker-compose logs -f
```

### 9.2 環境変数検証

実装フェーズで、起動前に環境変数を検証するスクリプトを作成します。

**検証項目**:
- `WP_SERVER`が`apache`または`fpm`であること
- `DB_TYPE`が`mysql`または`mariadb`であること
- `PHPMYADMIN_ENABLED`が`true`または`false`であること
- ポート番号が有効範囲内であること

---

## 10. 運用設計

### 10.1 起動・停止手順

#### 10.1.1 標準的な起動手順

```bash
# 1. .envファイル作成
cp .env.example .env

# 2. 必要に応じて.envを編集
vim .env

# 3. コンテナ起動
# Apache使用時
docker-compose up -d

# Nginx使用時
docker-compose --profile nginx up -d

# phpMyAdmin有効時
docker-compose --profile phpmyadmin up -d

# Nginx + phpMyAdmin
docker-compose --profile nginx --profile phpmyadmin up -d
```

#### 10.1.2 停止・削除手順

```bash
# コンテナ停止
docker-compose stop

# コンテナ停止・削除（ボリュームは保持）
docker-compose down

# コンテナ・ボリューム・ネットワーク削除
# 警告: すべてのデータが削除されます
docker-compose down -v
```

### 10.2 データバックアップ・リストア

#### 10.2.1 データベースバックアップ

```bash
# バックアップ
docker-compose exec db mysqldump -u wordpress -pwordpress wordpress > backup.sql

# リストア
docker-compose exec -T db mysql -u wordpress -pwordpress wordpress < backup.sql
```

#### 10.2.2 wp-contentバックアップ

```bash
# バックアップ
tar -czf wp-content-backup.tar.gz wordpress/wp-content/

# リストア
tar -xzf wp-content-backup.tar.gz
```

### 10.3 バージョン更新手順

#### 10.3.1 WordPressバージョン更新

```bash
# 1. .envでバージョン変更
# WP_VERSION=6.4 → WP_VERSION=6.5

# 2. コンテナ再作成
docker-compose up -d --force-recreate wordpress

# 3. ブラウザでWordPressにアクセスし、データベース更新を実行
```

#### 10.3.2 データベースバージョン更新

```bash
# 1. データベースバックアップ
docker-compose exec db mysqldump -u wordpress -pwordpress wordpress > backup.sql

# 2. .envでバージョン変更
# DB_VERSION=8.0 → DB_VERSION=8.4

# 3. コンテナ再作成
docker-compose up -d --force-recreate db

# 4. 動作確認
```

---

## 11. 構成パターン別設計

### 11.1 Pattern 1: Apache + MySQL

**用途**: 標準的なLAMP環境

**起動コマンド**:
```bash
docker-compose up -d
```

**.env設定**:
```bash
WP_SERVER=apache
DB_TYPE=mysql
DB_VERSION=8.0
PHPMYADMIN_ENABLED=false
```

**起動コンテナ**:
- wordpress (apache)
- db (mysql)

### 11.2 Pattern 2: Apache + MariaDB

**用途**: MariaDB使用環境

**起動コマンド**:
```bash
docker-compose up -d
```

**.env設定**:
```bash
WP_SERVER=apache
DB_TYPE=mariadb
DB_VERSION=11.4
PHPMYADMIN_ENABLED=false
```

**起動コンテナ**:
- wordpress (apache)
- db (mariadb)

### 11.3 Pattern 3: Nginx + MySQL

**用途**: Nginx + MySQL環境（高パフォーマンス）

**起動コマンド**:
```bash
docker-compose --profile nginx up -d
```

**.env設定**:
```bash
WP_SERVER=fpm
DB_TYPE=mysql
DB_VERSION=8.0
PHPMYADMIN_ENABLED=false
```

**起動コンテナ**:
- wordpress (fpm)
- nginx
- db (mysql)

### 11.4 Pattern 4: Nginx + MariaDB + phpMyAdmin

**用途**: フル機能環境

**起動コマンド**:
```bash
docker-compose --profile nginx --profile phpmyadmin up -d
```

**.env設定**:
```bash
WP_SERVER=fpm
DB_TYPE=mariadb
DB_VERSION=11.4
PHPMYADMIN_ENABLED=true
```

**起動コンテナ**:
- wordpress (fpm)
- nginx
- db (mariadb)
- phpmyadmin

---

## 12. 技術選定理由

### 12.1 Docker Compose選定理由

| 項目 | 理由 |
|-----|------|
| **シンプルさ** | 1つのYAMLファイルで複数コンテナを管理 |
| **再現性** | 同じ環境をチーム全員が構築可能 |
| **依存関係管理** | コンテナ間の起動順序を自動制御 |
| **ネットワーク自動構成** | サービス名でコンテナ間通信が可能 |

### 12.2 WordPress公式イメージ選定理由

| 項目 | 理由 |
|-----|------|
| **信頼性** | WordPress公式が提供 |
| **バリエーション** | Apache/FPM、複数PHPバージョンに対応 |
| **設定の簡潔さ** | 環境変数でWordPress設定が完結 |
| **メンテナンス** | 定期的なアップデート |

### 12.3 MySQL/MariaDB両対応の理由

| 項目 | 理由 |
|-----|------|
| **本番環境対応** | 本番環境と同じDBでテスト可能 |
| **互換性テスト** | 両DBでの動作確認が可能 |
| **柔軟性** | プロジェクトの要件に応じて選択 |

### 12.4 phpMyAdmin選定理由

| 項目 | 理由 |
|-----|------|
| **視覚的管理** | GUIでデータベースを操作 |
| **開発効率** | SQL実行、データ確認が容易 |
| **デバッグ支援** | テーブル構造やデータの確認 |
| **オプション化** | 不要な場合は無効化可能 |

---

## 13. 代替案と検討事項

### 13.1 検討した代替案

#### 13.1.1 Kubernetes使用

**メリット**:
- 本番環境に近い構成
- スケーラビリティ

**デメリット**:
- ローカル環境には過剰
- 学習コスト高
- セットアップ複雑

**結論**: ローカル開発環境にはDocker Composeが最適

#### 13.1.2 Webサーバーの固定（Apacheのみ等）

**メリット**:
- 設定がシンプル
- 選択肢が少なく迷わない

**デメリット**:
- 本番環境との差異が発生する可能性
- パフォーマンス比較ができない

**結論**: 柔軟な切り替えをサポート

#### 13.1.3 phpMyAdminの代わりにAdminer

**Adminerのメリット**:
- 軽量（単一PHPファイル）
- 複数DBに対応（PostgreSQL等）

**phpMyAdminのメリット**:
- 知名度が高い
- WordPress開発者に馴染み深い
- 機能が豊富

**結論**: 知名度と機能性からphpMyAdminを採用

### 13.2 今後の拡張可能性

実装後、以下の機能追加を検討可能です。

- **WP-CLI統合**: コマンドラインでのWordPress操作
- **Xdebug統合**: PHPデバッグ機能
- **Mailhog統合**: メール送信テスト
- **Redis統合**: オブジェクトキャッシュ
- **HTTPS対応**: mkcert等を使用したローカルSSL

---

## 14. 次フェーズへの引き継ぎ

### 14.1 実装計画フェーズで検討すべき項目

1. **docker-compose.ymlの詳細実装**
   - Apache/FPM切り替えの実装方法（override.yml使用等）
   - profiles機能の詳細設定
   - ヘルスチェックの詳細設定

2. **起動スクリプトの作成**
   - 環境変数を読み取り、適切なprofileで起動するスクリプト
   - 環境変数検証スクリプト

3. **nginx.confの作成**
   - 上記設計に基づいた実際の設定ファイル

4. **ディレクトリ初期化**
   - `wordpress/wp-content`ディレクトリの作成
   - `nginx`ディレクトリの作成

5. **README.mdの作成**
   - セットアップ手順
   - トラブルシューティング
   - 各構成パターンの説明

6. **テスト計画**
   - 各構成パターンでの動作確認
   - エラーケースのテスト

---

## 15. 変更履歴

| バージョン | 日付 | 変更内容 | 変更者 |
|----------|------|---------|--------|
| 1.0 | 2026-02-17 | 初版作成 | Claude Code |

---

## 16. 承認

| 役割 | 氏名 | 承認日 | 署名 |
|-----|------|--------|------|
| 設計者 | Claude Code | 2026-02-17 | - |
| レビュアー | - | - | - |
| 承認者 | - | - | - |

---

**このドキュメントに対するフィードバックをお願いいたします。**
