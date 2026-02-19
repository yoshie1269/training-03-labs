# LAMP＋WordPress 2台構成 手順書（Web＋DB分離構成）

---

## この構成でしていること
- Webサーバ（Apache＋PHP＋WordPress）とDBサーバ（MariaDB）を分離する
- DBをPrivate側に配置し、外部公開を防ぐ
- セキュリティと可用性を向上させる

構成イメージ：
```bash
[Internet]
    ↓ 80/443
[EC2-Web]  →  TCP/3306  →  [EC2-DB]
 Apache/PHP/WP              MariaDB
```

---

## 1. セキュリティグループ設計（最重要）

---

### この工程でしていること
- 通信経路を最小限に制限する
- DBを外部公開しない設計にする

### SG-Web（Web用）
```bash
Inbound
22  → 自分IP（SSH）
80  → 0.0.0.0/0（HTTP）
```

### SG-DB（DB用）
```bash
Inbound
22    → 自分IP
3306  → SG-Web（IPではなくSG指定）
```
OKの目安：
- SG-DBの3306がSG-Web指定になっている
- 0.0.0.0/0で3306を開けていない

---

## 2. DBサーバ構築（EC2-DB）

---

### 2-1 MariaDB インストール

#### この工程でしていること
- DB専用サーバを構築する

#### 実行コマンド
```bash
sudo dnf install mariadb105-server -y
```

#### 確認
```bash
systemctl status mariadb
```

OKの目安：
- Loaded と表示される（まだactiveでなくてもOK）

---

### 2-2 MariaDB 起動・自動化

#### 実行コマンド
```bash
sudo systemctl start mariadb
```
```bash
sudo systemctl enable mariadb
```

#### 確認
```bash
systemctl status mariadb
```
```bash
sudo systemctl is-enabled mariadb
```

OKの目安：
- active (running)
- enabled

---

### 2-3 初期セキュリティ設定

#### 実行コマンド
```bash
sudo mysql_secure_installation
```

#### この工程でしていること
- rootパスワード設定
- 匿名ユーザ削除
- リモートroot禁止
- testDB削除

OKの目安：
- すべてYで完了

---

### 2-4 WordPress用DB作成

#### この工程でしていること
- WordPress専用DBと接続ユーザを作成する

#### 実行コマンド
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

#### 確認
```bash
SHOW DATABASES;
```
```bash
SELECT user, host FROM mysql.user;
```
```bash
SHOW GRANTS FOR 'wpuser'@'%';
```

OKの目安：
- wordpress_db が存在する
- wpuser が存在する
- 権限が付与されている

---

## 3. Webサーバ構築（EC2-Web）
※webサーバで実行

### 3-1 LAMP導入

#### この工程でしていること
- Apache＋PHP環境を構築する

#### 実行コマンド
```bash
sudo dnf upgrade -y
```
```bash
sudo dnf install httpd php php-mysqlnd php-fpm wget php-gd -y
```

#### 確認
```bash
rpm -qa | grep httpd
```
```bash
rpm -qa | grep php
```
OKの目安：
- httpd、phpパッケージが表示される

---

### 3-2 Apache起動
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```
確認：
```bash
systemctl status httpd
```
OKの目安：
- active (running)

---

### 3-3 WordPress DL・展開
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```
確認：
```bash
ls
```
OKの目安：
- wordpress ディレクトリがある

---

### 3-4 配置
```bash
sudo cp -r wordpress/* /var/www/html/
```
確認：
```bash
ls /var/www/html
```
OKの目安：
- wp-admin などが表示される

---

### 3-5 権限設定
```bash
sudo chown -R apache:apache /var/www
```
```bash
sudo chmod -R 755 /var/www
```
確認：
```bash
ls -l /var/www
```

OKの目安：
- 所有者が apache
- パーミッションが755

---

## 4. WordPress 設定（DB接続）

### この工程でしていること
- WebサーバからDBへ接続できるようにする

wp-config.php 設定：
```bash
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wpuser');
define('DB_PASSWORD', 'StrongPass123');
define('DB_HOST', 'DBサーバのPrivateIP');
```
確認：
```bash
php -l /var/www/html/wp-config.php
```
OKの目安：
- No syntax errors detected

---

## 5. Web→DB 疎通確認
```bash
mysql -u wpuser -h DBのPrivateIP -p wordpress_db
```
OKの目安：
- MariaDBモニタに入れる

---

## 6. Apache設定
```bash
vi /etc/httpd/conf/httpd.conf
```
```bash
<Directory "/var/www/html">
AllowOverride All
</Directory>
```
確認：
```bash
httpd -t
```
OKの目安：
- Syntax OK

再起動：
```bash
systemctl restart httpd
```

---

## 7. Webインストール

ブラウザ：
```bash
http://WebのPublicIPアドレス/
```
OKの目安：
- WordPressセットアップ画面表示

---

## 8. 最終確認
```bash
systemctl status httpd
```
```bash
systemctl status mariadb
```
OKの目安：
- 両方 active (running)

ポート確認：
```bash
ss -lnt | grep 80
```
```bash
ss -lnt | grep 3306
```
---

## 9. 典型ミス対策

DB接続エラー → SG 3306未開放
Timeout → SG/VPCルート確認
接続不可 → DB_HOST間違い
403 → 権限設定ミス
白画面 → PHP不足

---

## 完成状態

- Web/DB分離
- DBはPrivate配置
- SG最小権限
- WordPress正常稼働