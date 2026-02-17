# WordPress Docker環境

Docker Composeを使用したWordPress開発環境です。Webサーバー（Apache/Nginx）、データベース（MySQL/MariaDB）、phpMyAdminを環境変数で柔軟に切り替えられます。

## 特徴

- ✅ **柔軟な構成**: Apache/Nginx、MySQL/MariaDBを環境変数で簡単切り替え
- ✅ **バージョン管理**: WordPress、PHP、データベースのバージョンを自由に指定可能
- ✅ **データ永続化**: wp-contentとデータベースデータを永続化
- ✅ **開発最適化**: デバッグモード、エラーログ出力
- ✅ **管理ツール**: phpMyAdmin（オプション）
- ✅ **簡単セットアップ**: .envファイルで一元管理

## 必須要件

- Docker: 20.10以上
- Docker Compose: 2.0以上

## クイックスタート

### 1. 環境設定ファイルの作成

```bash
cp .env.example .env
```

### 2. 環境変数の編集（オプション）

`.env`ファイルを開き、必要に応じて設定を変更します。

```bash
# 例: Nginxを使用する場合
WP_SERVER=fpm

# 例: MariaDBを使用する場合
DB_TYPE=mariadb
DB_VERSION=10.11

# 例: phpMyAdminを有効化する場合
PHPMYADMIN_ENABLED=true
```

### 3. コンテナの起動

#### パターン1: Apache + MySQL（デフォルト）

```bash
docker-compose up -d
```

#### パターン2: Nginx + MySQL

```bash
# .envファイルでWP_SERVER=fpmに設定
docker-compose --profile nginx up -d
```

#### パターン3: phpMyAdminを含める場合

```bash
# .envファイルでPHPMYADMIN_ENABLED=trueに設定
docker-compose --profile phpmyadmin up -d

# NginxとphpMyAdmin両方を使う場合
docker-compose --profile nginx --profile phpmyadmin up -d
```

### 4. WordPressへアクセス

ブラウザで以下のURLを開きます。

- WordPress: http://localhost:8080
- phpMyAdmin（有効時）: http://localhost:8081

### 5. WordPressの初期セットアップ

初回アクセス時にWordPressのセットアップ画面が表示されます。

1. 言語を選択
2. サイト情報を入力
3. 管理者アカウントを作成

## 構成パターン

### パターン1: Apache + MySQL（最も一般的）

```env
WP_SERVER=apache
DB_TYPE=mysql
DB_VERSION=8.0
```

```bash
docker-compose up -d
```

### パターン2: Apache + MariaDB

```env
WP_SERVER=apache
DB_TYPE=mariadb
DB_VERSION=10.11
```

```bash
docker-compose up -d
```

### パターン3: Nginx + MySQL

```env
WP_SERVER=fpm
DB_TYPE=mysql
DB_VERSION=8.0
```

```bash
docker-compose --profile nginx up -d
```

### パターン4: Nginx + MariaDB

```env
WP_SERVER=fpm
DB_TYPE=mariadb
DB_VERSION=11.4
```

```bash
docker-compose --profile nginx up -d
```

## 環境変数一覧

### WordPress設定

| 変数名 | 説明 | デフォルト値 | 例 |
|--------|------|--------------|-----|
| `WP_VERSION` | WordPressバージョン | `latest` | `6.4`, `6.3` |
| `PHP_VERSION` | PHPバージョン | `8.2` | `8.3`, `8.1` |
| `WP_SERVER` | Webサーバー種類 | `apache` | `fpm` (Nginx) |
| `WP_PORT` | アクセスポート | `8080` | `8000`, `9000` |
| `WP_DEBUG` | デバッグモード | `1` | `0` (無効) |
| `WP_DEBUG_LOG` | デバッグログ出力 | `1` | `0` (無効) |
| `WP_DEBUG_DISPLAY` | デバッグ画面表示 | `1` | `0` (無効) |

### データベース設定

| 変数名 | 説明 | デフォルト値 | 例 |
|--------|------|--------------|-----|
| `DB_TYPE` | データベース種類 | `mysql` | `mariadb` |
| `DB_VERSION` | データベースバージョン | `8.0` | `10.11` (MariaDB) |
| `MYSQL_DATABASE` | データベース名 | `wordpress` | 任意の名前 |
| `MYSQL_USER` | データベースユーザー | `wordpress` | 任意のユーザー名 |
| `MYSQL_PASSWORD` | データベースパスワード | `wordpress_password` | 強力なパスワード |
| `MYSQL_ROOT_PASSWORD` | rootパスワード | `root_password` | 強力なパスワード |

### phpMyAdmin設定

| 変数名 | 説明 | デフォルト値 | 例 |
|--------|------|--------------|-----|
| `PHPMYADMIN_ENABLED` | 有効化フラグ | `false` | `true` |
| `PHPMYADMIN_VERSION` | phpMyAdminバージョン | `latest` | `5.2` |
| `PHPMYADMIN_PORT` | アクセスポート | `8081` | `8082` |

## よく使うコマンド

### コンテナの起動

```bash
# 通常起動
docker-compose up -d

# Nginxプロファイル付き起動
docker-compose --profile nginx up -d

# phpMyAdminプロファイル付き起動
docker-compose --profile phpmyadmin up -d

# 両方のプロファイル付き起動
docker-compose --profile nginx --profile phpmyadmin up -d
```

### コンテナの停止

```bash
docker-compose down
```

### コンテナの再起動

```bash
docker-compose restart
```

### ログの確認

```bash
# すべてのコンテナのログ
docker-compose logs

# 特定のコンテナのログ
docker-compose logs wordpress
docker-compose logs db

# リアルタイムでログを表示
docker-compose logs -f
```

### コンテナの状態確認

```bash
docker-compose ps
```

### コンテナ内でコマンド実行

```bash
# WordPressコンテナに入る
docker-compose exec wordpress bash

# データベースコンテナに入る
docker-compose exec db bash

# MySQLクライアントでデータベースに接続
docker-compose exec db mysql -u wordpress -p
```

## データ管理

### データのバックアップ

#### wp-contentのバックアップ

```bash
tar -czf wp-content-backup-$(date +%Y%m%d).tar.gz wordpress/wp-content/
```

#### データベースのバックアップ

```bash
docker-compose exec db mysqldump -u wordpress -pwordpress_password wordpress > backup-$(date +%Y%m%d).sql
```

### データのリストア

#### wp-contentのリストア

```bash
tar -xzf wp-content-backup-20260217.tar.gz
```

#### データベースのリストア

```bash
docker-compose exec -T db mysql -u wordpress -pwordpress_password wordpress < backup-20260217.sql
```

### データの完全削除

```bash
# コンテナとボリュームを削除
docker-compose down -v

# wp-contentも削除する場合
rm -rf wordpress/wp-content/*
```

## バージョンアップ

### WordPressのバージョンアップ

1. `.env`ファイルの`WP_VERSION`を変更
2. コンテナを再作成

```bash
docker-compose up -d --force-recreate wordpress
```

### データベースのバージョンアップ

**注意**: データベースのバージョンアップ前に必ずバックアップを取得してください。

1. データベースをバックアップ
2. `.env`ファイルの`DB_VERSION`を変更
3. コンテナを再作成

```bash
docker-compose up -d --force-recreate db
```

## トラブルシューティング

### WordPressが起動しない

**症状**: `http://localhost:8080`にアクセスできない

**対処方法**:

1. コンテナの状態を確認

```bash
docker-compose ps
```

2. ログを確認

```bash
docker-compose logs wordpress
```

3. データベースの接続を確認

```bash
docker-compose logs db
```

### データベース接続エラー

**症状**: 「データベース接続確立エラー」が表示される

**対処方法**:

1. データベースコンテナが起動しているか確認

```bash
docker-compose ps db
```

2. データベースの認証情報を確認

```bash
# .envファイルの設定を確認
cat .env | grep MYSQL
```

3. データベースコンテナを再起動

```bash
docker-compose restart db
```

### ポート競合エラー

**症状**: `port is already allocated`エラー

**対処方法**:

1. 使用されているポートを確認

```bash
# macOS/Linux
lsof -i :8080

# Windows
netstat -ano | findstr :8080
```

2. `.env`ファイルで別のポートに変更

```env
WP_PORT=8000
PHPMYADMIN_PORT=8001
```

### phpMyAdminにアクセスできない

**症状**: `http://localhost:8081`にアクセスできない

**対処方法**:

1. phpMyAdminプロファイルで起動しているか確認

```bash
docker-compose --profile phpmyadmin up -d
```

2. コンテナの状態を確認

```bash
docker-compose ps phpmyadmin
```

### パーミッションエラー

**症状**: wp-contentへの書き込みができない

**対処方法**:

```bash
# wp-contentのパーミッションを変更
sudo chmod -R 777 wordpress/wp-content/
```

### Nginx使用時にWordPressが表示されない

**対処方法**:

1. Nginxプロファイルで起動しているか確認

```bash
docker-compose --profile nginx ps
```

2. nginx.confの設定を確認

```bash
docker-compose exec nginx cat /etc/nginx/nginx.conf
```

3. Nginxのログを確認

```bash
docker-compose logs nginx
```

## ディレクトリ構造

```
.
├── docker-compose.yml       # Docker Compose設定ファイル
├── .env.example             # 環境変数のテンプレート
├── .env                     # 環境変数（Git管理外）
├── .gitignore               # Git除外設定
├── nginx/
│   └── nginx.conf           # Nginx設定ファイル
├── wordpress/
│   └── wp-content/          # WordPressコンテンツ（プラグイン、テーマ、アップロード）
└── README.md                # このファイル
```

## セキュリティ注意事項

⚠️ **この環境はローカル開発専用です。本番環境では使用しないでください。**

本番環境で使用する場合は、以下の対策が必要です。

- SSL/TLS証明書の設定
- 強力なパスワードの使用
- データベースのrootアクセス制限
- phpMyAdminの無効化または適切な認証設定
- ファイアウォールの設定
- 定期的なバックアップ

## ライセンス

このプロジェクトはMITライセンスの下で公開されています。

## 参考リンク

- [WordPress公式サイト](https://wordpress.org/)
- [WordPress Dockerイメージ](https://hub.docker.com/_/wordpress)
- [MySQL Dockerイメージ](https://hub.docker.com/_/mysql)
- [MariaDB Dockerイメージ](https://hub.docker.com/_/mariadb)
- [phpMyAdmin Dockerイメージ](https://hub.docker.com/_/phpmyadmin)
- [Nginx公式サイト](https://nginx.org/)