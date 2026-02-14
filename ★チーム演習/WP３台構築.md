# WordPress 三層構成 構築手順書（超詳細版）

構成：
Web（Apache） → AP（PHP-FPM / WordPress） → DB（MariaDB）

---

# 0. 事前理解（必ず読む）

この構成では：

- Web：画面表示担当
- AP：PHP処理担当
- DB：データ保存担当

WebサーバではPHPは動かさない。
必ずAPに処理を渡す。

---

# 第1章 Webサーバ構築（web-1）

---

## 1-1. Apacheインストール

目的：
Web公開のためのソフトを入れる。
```bash
dnf update -y
```
```bash
dnf install httpd -y
```
systemctl start httpd
```bash
systemctl enable httpd
```
```bash
systemctl status httpd
```
---

## 1-2. Apache起動確認

ブラウザで確認：

http://WebのIP

OK基準：
→ 「It works!」と表示される

表示されない場合：
→ SG / Apache未起動を疑う

---

## 1-3. php-fpm導入（連携用）

目的：
APと連携するための設定ファイルを用意する。
```bash
dnf install php-fpm -y
```
---

## 1-4. PHP連携設定
```bash
vi /etc/httpd/conf.d/php.conf
```
修正内容：
```bash
<FilesMatch \.(php|phar)$>
SetHandler "proxy:fcgi://ap-1.wp.local:8000"
</FilesMatch>
```
注意：
unix:/run/php-fpm.sock が残っていたら必ず削除

---

## 1-5. Apache再起動
```bash
systemctl restart httpd
```
念のため停止：
```bash
systemctl stop php-fpm
```
理由：
Web側でPHPを動かさないため

---

# 第2章 APサーバ構築（ap-1 / ap-2）

---

## 2-1. PHP・MariaDBツール導入

目的：
PHP実行＋DB操作準備
```bash
dnf install mariadb105-server php php-fpm php-mysqli php-json php-devel -y
```
---

## 2-2. php-fpm起動
```bash
systemctl start php-fpm
```
```bash
systemctl enable php-fpm
```
```bash
systemctl status php-fpm
```
OK基準：
→ active (running)

---

## 2-3. php-fpm待受設定（最重要）
```bash
vi /etc/php-fpm.d/www.conf
```
以下を探す：
```bash
;listen =
;listen.allowed_clients =
```
修正：
```bash
listen = ap-1.wp.local:8000
listen.allowed_clients = web-1.wp.local
```
※「;」を必ず消す

---

## 2-4. 再起動
```bash
systemctl restart php-fpm
```
---

## 2-5. phpinfo作成

目的：
PHP連携確認用

echo "<?php phpinfo(); ?>" > /var/www/html/phpinfo.php

chown apache:apache /var/www/html/phpinfo.php

---

## 2-6. 表示確認

ブラウザ：

http://WebのIP/phpinfo.php

OK基準：
→ PHP情報画面が出る

出ない場合：
→ Web/AP設定を再確認

---

# 第3章 DBサーバ構築（db-1）

---

## 3-1. MariaDB導入

dnf update -y

dnf install mariadb105-server -y

systemctl start mariadb
systemctl enable mariadb

systemctl status mariadb

---

## 3-2. 初期設定

mysql_secure_installation

全項目：Y 推奨

---

## 3-3. DB・User作成

mysql -u root -p

CREATE DATABASE wordpress_db;

CREATE USER 'wordpress-user'@'ap-1.wp.local'
IDENTIFIED BY 'wordpress-password';

GRANT ALL ON wordpress_db.*
TO 'wordpress-user'@'ap-1.wp.local';

FLUSH PRIVILEGES;

exit

---

## 3-4. 接続確認（AP → DB）

APで実施：

mysql -u wordpress-user -h db-1.wp.local -p wordpress_db

OK基準：
→ 接続成功

---

# 第4章 WordPress構築（AP）

---

## 4-1. ダウンロード

cd /usr/local

wget https://wordpress.org/latest.tar.gz

tar -xzf latest.tar.gz

---

## 4-2. 設定ファイル準備

cp wordpress/wp-config-sample.php wordpress/wp-config.php

---

## 4-3. 配置

mv wordpress/* /var/www/html/

---

## 4-4. DB接続設定

vi /var/www/html/wp-config.php

変更：

define('DB_NAME','wordpress_db');
define('DB_USER','wordpress-user');
define('DB_PASSWORD','wordpress-password');
define('DB_HOST','db-1.wp.local');

---

## 4-5. 接続確認

mysql -u wordpress-user -h db-1.wp.local -p wordpress_db

---

# 第5章 WordPress構築（Web）

※APと同じ作業を行う

理由：
両方にWPが必要な構成のため

---

cd /usr/local

wget https://wordpress.org/latest.tar.gz

tar -xzf latest.tar.gz

mv wordpress/* /var/www/html/

---

# 第6章 最終確認

---

## 6-1. 全サーバ再起動

reboot

---

## 6-2. 表示確認

ブラウザ：

http://WebのIP

OK基準：
→ WordPress初期画面

---

ログイン情報：

User：wordpress-user  
Pass：wordpress-password

---

# 7. よくある失敗集

❌ phpinfo出ない  
→ php.conf / www.conf ミス

❌ DB接続エラー  
→ DB_HOST / 権限ミス

❌ 真っ白画面  
→ Apacheログ確認

---

# 8. 完了条件

以下すべてOK：

☑ It works 表示  
☑ phpinfo 表示  
☑ WP初期画面  
☑ DB接続成功  

→ 完成

---