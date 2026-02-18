# LAMP＋WordPress 2台構成 手順書（Web＋DB分離構成）

---

## 概要

本手順書では、以下の構成で WordPress 環境を構築する。

- Webサーバ：Apache + PHP + WordPress
- DBサーバ：MariaDB
- 両サーバを分離してセキュリティ・可用性を向上させる

---

## 構成概要
```bash
[Internet]
   |
   | HTTP/HTTPS
   v
[EC2-Web] -------- TCP/3306 --------> [EC2-DB]
 Apache/PHP/WP                     MariaDB
```

- Web：Public Subnet
- DB：Private Subnet（推奨）
- DB は外部公開しない

---

## 前提条件

| 項目 | 内容 |
|------|------|
| Webサーバ | Amazon Linux 2023 |
| DBサーバ | Amazon Linux 2023 |
| 台数 | EC2×2台 |
| WordPress | 最新版 |
| DB | MariaDB |

---

## 1. セキュリティグループ設計（最重要）

---

### Web用SG（SG-Web）

Inbound

| Port | Source | 用途 |
|------|--------|------|
| 22 | 自分IP | SSH |
| 80 | 0.0.0.0/0 | HTTP |

---

### DB用SG（SG-DB）

Inbound

| Port | Source | 用途 |
|------|--------|------|
| 22 | 自分IP | SSH |
| 3306 | SG-Web | DB接続 |

※ IP直指定よりSG指定

---

## 2. DBサーバ構築（EC2-DB）

---

## 2-1. MariaDB インストール

### この工程でしていること
- DB専用サーバを構築する

### 実行コマンド
```bash
sudo dnf install mariadb105-server -y
```

### 確認
```bash
systemctl status mariadb
```
---

## 2-2. MariaDB 起動・自動化
```bash
sudo systemctl start mariadb
```
```bash
sudo systemctl enable mariadb
```

---

## 2-3. 初期セキュリティ設定
```bash
sudo mysql_secure_installation
```

設定：
- rootパスワード設定 → Y
- 匿名削除 → Y
- リモートroot禁止 → Y
- test削除 → Y
- reload → Y

---

## 2-4. WordPress用DB作成

```bash
mysql -u root -p
```
```bash
CREATE DATABASE wordpress_db;
```
```bash
CREATE USER 'wpuser'@'%' IDENTIFIED BY 'StrongPass123';
```
```bash
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wpuser'@'%';
```
```bash
FLUSH PRIVILEGES;
```
```bash
exit
```

---

## 3. Webサーバ構築（EC2-Web）

---

## 3-1. LAMP導入（DB除く）
```bash
sudo dnf upgrade -y
```
```bash
sudo dnf install httpd php php-mysqlnd php-fpm wget php-gd -y
```

---

## 3-2. Apache 起動
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```
---

## 3-3. WordPress DL・展開
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```
---

## 3-4. 配置
```bash
sudo cp -r wordpress/* /var/www/html/
```
---

## 3-5. 権限設定
```bash
sudo chown -R apache:apache /var/www
```
```bash
sudo chmod -R 755 /var/www
```

---

## 4. WordPress 設定（DB接続）

---

## 4-1. wp-config 作成
```bash
cd wordpress
```
```bash
cp wp-config-sample.php wp-config.php
```

---

## 4-2. DB接続設定
```bash
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'StrongPass123');
define('DB_HOST', 'DBサーバのPrivateIP');
```
---

## 4-3. SALT設定

`https://api.wordpress.org/secret-key/1.1/salt/`

生成して貼付

---

## 5. Web→DB 疎通確認

---

### DB接続テスト
```bash
mysql -u wpuser -h DBのPrivateIP -p wordpress_db
```
成功すればOK

---

## 6. Apache 設定

### AllowOverride
```bash
sudo vi /etc/httpd/conf/httpd.conf
```
```bash
<Directory "/var/www/html">
AllowOverride All
</Directory>
```

---

### 再起動
```bash
sudo systemctl restart httpd
```
---

## 7. Webインストール

---

ブラウザ：

`http://WebのPublicDNS/`

WordPressセットアップ画面表示

---

## 8. 最終確認

---

### サービス
```bash
systemctl status httpd
```
```bash
systemctl status mariadb
```

---

### ポート

Web：
```bash
netstat -ln | grep 80
```
DB：
```bash
netstat -ln | grep 3306
```
---

### 疎通

Web→DB：
telnet DB-IP 3306

---

## 9. 典型ミス対策（超重要）

| 症状 | 原因 |
|------|------|
| DB接続エラー | SG 3306未開放 |
| Timeout | VPC/RT/SGミス |
| 接続不可 | bind-address |
| 403 | 権限ミス |
| 白画面 | PHP不足 |

---

## 完成状態

- Web/DB分離
- SG最小権限
- 安全構成
- WordPress稼働
