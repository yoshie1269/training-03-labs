# DB LAMP サーバー構築 手順書

# ■ db-1：Apache / PHP / MariaDB 導入

## 概要

本手順書では、Amazon Linux 2023 上に LAMP 環境を構築するため、  
以下の作業を実施する。

- OS・パッケージの更新
- Apache / PHP / MariaDB の導入
- Apache の起動・自動起動設定
- Web ディレクトリ権限の設定
- PHP 動作確認
- MariaDB 初期セキュリティ設定

---

## 前提条件

- Amazon Linux 2023 を使用していること
- EC2 にパブリックアクセス可能であること
- セキュリティグループで 80/TCP が開放されていること
- root 権限または sudo 権限を持っていること

---

## 1-1. OS・パッケージを最新化する


### この工程でしていること
- OS の更新を行い、脆弱性を解消する
- 依存関係の問題を防止する


### 実行コマンド
```bash
sudo dnf upgrade -y
```


### コマンドの意味
- dnf upgrade：パッケージ更新
- -y：確認なしで実行


### 確認方法とOKの目安
- エラーなく処理が完了する
- Complete! が表示される


---

## 1-2. Apache / PHP / 関連モジュールをインストールする


### この工程でしていること
- Web サーバ（Apache）を導入する
- PHP 実行環境を構築する
- WordPress 等に必要な拡張を導入する


### 実行コマンド
```bash
sudo dnf install -y httpd wget php-fpm php-mysqli php-json php php-devel
```


### コマンドの意味

| 項目 | 内容 |
|------|------|
| httpd | Apache 本体 |
| wget | ダウンロード用 |
| php-fpm | PHP 実行管理 |
| php-mysqli | DB連携 |
| php-json | JSON処理 |
| php-devel | 開発用 |


### 確認方法とOKの目安
```bash
dnf list installed | grep httpd
```
```bash
dnf list installed | grep php
```

- パッケージが表示される

---

## 1-3. MariaDB をインストールする

### この工程でしていること
- データベースサーバ（MariaDB）を導入する
- WordPress などのデータ保存先を準備する


### 実行コマンド
```bash
sudo dnf install mariadb105-server
```
```bash
sudo dnf info mariadb105-server
```


### コマンドの意味
■ mariadb105-server
- MariaDB 10.5 系のデータベースサーバ本体

■ dnf install
- パッケージをインストールする

■ dnf info
- インストール済みパッケージの詳細情報を表示する



### 確認コマンド

■ パッケージが正しく入ったか確認
```bash
rpm -qa | grep mariadb
```

OKの目安：
- mariadb105-server が表示される

■ 詳細情報確認
```bash
dnf list installed | grep mariadb
```
OKの目安：
- mariadb105-server.x86_64 などが表示される

■ サービス存在確認
```bash
systemctl list-unit-files | grep mariadb
```
OKの目安：
- mariadb.service が表示される

■ バージョン確認
```bash
mysql --version
```
OKの目安：
- mysql  Ver 15.1 Distrib 10.5.xx-MariaDB などと表示される

---

## 1-4. Apache を起動・自動起動設定する


### この工程でしていること
- Web サーバを起動する
- OS 起動時に自動起動させる


### 実行コマンド
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```


### コマンドの意味
- start：起動
- enable：自動起動登録


### 確認方法とOKの目安
```bash
sudo systemctl status httpd
```
```bash
sudo systemctl is-enabled httpd
```
- active (running)
- enabled と表示される

---

## 1-5. ec2-user を Apache グループへ追加する


### この工程でしていること
- ec2-user が Web ファイルを編集できるようにする
- 権限管理を適切にする


### 実行コマンド
```bash
sudo usermod -a -G apache ec2-user
```
```bash
exit
```
 - sshでログイン


### コマンドの意味
- usermod -a -G：グループ追加
- exit：再ログイン
- groups：所属グループ表示


### 確認方法とOKの目安
```bash
groups
```
- apache が表示される

---

## 1-6. Web ディレクトリの権限を設定する


### この工程でしていること
- Apache と ec2-user が共同管理できるように設定する
- セキュリティを維持しつつ編集可能にする


### 実行コマンド
```bash
sudo chown -R ec2-user:apache /var/www
```
```bash
sudo find /var/www -type d -exec chmod 2775 {} \;
```
```bash
sudo find /var/www -type f -exec chmod 0664 {} \;
```


### コマンドの意味

| 項目 | 内容 |
|------|------|
| chown | 所有者変更 |
| chmod 2775 | SGID設定 |
| find | 再帰適用 |


### 確認方法とOKの目安
```bash
ls -ld /var/www
```
- ec2-user apache になっている

---

## 1-7. PHP 動作確認を行う


### この工程でしていること
- PHP が正常に動作するか確認する
- LAMP 構成の正常性を検証する


### 実行コマンド
```bash
echo '<?php phpinfo(); ?>' | sudo tee /var/www/html/phpinfo.php
```

### 確認方法

ブラウザでアクセス
`http://<パブリックIPアドレス>/phpinfo.php`


### OKの目安
- PHP 情報ページが表示される


### テストファイル削除
```bash
rm /var/www/html/phpinfo.php
```

---

## 1-8. MariaDB を起動する


### この工程でしていること
- データベースサービスを開始する


### 実行コマンド
```bash
sudo systemctl start mariadb
```


### 確認方法とOKの目安
```bash
systemctl status mariadb
```
- active (running)

---

## 1-9. MariaDB 初期セキュリティ設定を行う


### この工程でしていること
- DB を安全に運用できる状態にする
- 不正アクセスを防止する


### 実行コマンド
```bash
sudo mysql_secure_installation
```


### 設定内容（対話形式）

1. 現在の root パスワード  
   → Enter

2. root パスワード設定  
   → Y → 設定

3. 匿名ユーザー削除  
   → Y

4. リモート root 無効化  
   → Y

5. テストDB削除  
   → Y

6. 権限再読込  
   → Y


### 確認方法とOKの目安
- エラーなく完了
- All done! と表示される

---

## 1-10. MariaDB の自動起動設定


### この工程でしていること
- OS 起動時に DB を自動起動させる


### 実行コマンド
```bash
sudo systemctl enable mariadb
```


### 確認方法とOKの目安
```bash
systemctl is-enabled mariadb
```

- enabled

---

## 1-11. ここまでで完了していること

- Apache が稼働している
- PHP が動作している
- MariaDB が稼働している
- Web 権限が適切に設定されている
- 基本セキュリティが確保されている

---

# ■ db-2：WordPress 導入・公開設定

## 概要

本手順書では、構築済みの LAMP 環境上に WordPress を導入し、  
ブログとして公開できる状態まで設定する。

本工程により、以下を実現する。

- WordPress 本体の導入
- MariaDB への専用データベース作成
- WordPress 設定ファイル作成
- Apache 設定変更
- ファイル権限調整
- Web インストール準備完了


## 前提条件

- LAMP 環境が構築済みであること
- Apache / MariaDB が正常稼働していること
- /var/www/html が利用可能であること
- root 権限または sudo 権限があること
- ポート 80 が開放されていること

---

## 2-1. WordPress 用パッケージをインストールする


### この工程でしていること
- WordPress に必要な PHP 拡張・DB 連携機能を追加する
- 不足パッケージを補完する


### 実行コマンド
```bash
dnf install wget php-mysqlnd httpd php-fpm php-mysqli mariadb105-server php-json php php-devel -y
```


### コマンドの意味

| 項目 | 内容 |
|------|------|
| php-mysqlnd | PHP-DB連携 |
| php-fpm | PHP管理 |
| mariadb | DB |
| wget | DL |
| php-devel | 拡張用 |



### 確認方法とOKの目安
```bash
dnf list installed | grep php
```
```bash
dnf list installed | grep mariadb
```
- 必要パッケージが表示される


## 2-2. WordPress 本体をダウンロード・展開する


### この工程でしていること
- WordPress 最新版を取得する
- 利用可能な状態に展開する


### 実行コマンド
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```


### コマンドの意味
- wget：ダウンロード
- tar -xzf：gzip展開


### 確認方法とOKの目安
```bash
ls
```
- wordpress ディレクトリが存在


## 2-3. Apache / MariaDB を起動する


### この工程でしていること
- WordPress 利用に必要なサービスを起動する


### 実行コマンド
```bash
sudo systemctl start mariadb httpd
```


### 確認方法とOKの目安
```bash
systemctl status mariadb
```
```bash
systemctl status httpd
```
- active (running)

---

## 2-4. WordPress 用データベースを作成する


### この工程でしていること
- WordPress 専用 DB とユーザーを作成する
- アクセス権限を付与する


### MariaDB へ接続
```bash
mysql -u root -p
```

### SQL 設定
```bash
CREATE USER 'wordpress-user'@'localhost' IDENTIFIED BY '<パスワード>';
```
```bash
CREATE DATABASE wordpress_db;
```
```bash
GRANT ALL PRIVILEGES ON wordpress_db.* TO 'wordpress-user'@'localhost';
```
```bash
FLUSH PRIVILEGES;
```
```bash
exit
```


### 設定の意味

| 項目 | 内容 |
|------|------|
| CREATE USER | DBユーザー作成 |
| CREATE DATABASE | DB作成 |
| GRANT | 権限付与 |
| FLUSH | 反映 |



### 確認方法とOKの目安
```bash
SHOW DATABASES;
```
```bash
SELECT user,host FROM mysql.user;
```
- wordpress-db が存在
- wordpress-user が存在

---

## 2-5. wp-config.php を作成する（修正版：作成→配置の順）


### この工程でしていること
- 作業用ディレクトリ（例：~/wordpress）で wp-config.php を作成・編集する
- DB接続情報を正しく設定し、構文エラーを防ぐ
- 公開前に設定ミスを検知する



### 事前確認（作業場所の明確化）
```bash
ls -la ~/wordpress
```
```bash
ls -la ~/wordpress/wp-config-sample.php
```
OKの目安：
- wordpress ディレクトリが存在する
- wp-config-sample.php が存在する



### 実行内容
```bash
cd ~/
```
```bash
cp wordpress/wp-config-sample.php wordpress/wp-config.php
```


### 確認
```bash
ls wordpress/ | grep config
```
- wp-config.phpがある


### 実行内容
```bash
vi wordpress/wp-config.php
```

### 設定内容（DB作成時と完全一致）
```bash
define('DB_NAME', 'wordpress_db');
define('DB_USER', 'wordpress-user');
define('DB_PASSWORD', 'yoshie');
define('DB_HOST', 'localhost');
```


### 設定の意味

- DB_NAME：接続先データベース名
- DB_USER：DB接続ユーザー名
- DB_PASSWORD：DB接続パスワード
- DB_HOST：DBサーバの接続先（同一サーバなら localhost）


### 注意事項（障害防止）

- ハイフン（-）とアンダースコア（_）を混在させない
- DB_USER に @localhost を書かない
- 全角クォートを使わない
- 必ず半角シングルクォート（'）を使う


### セキュリティキー設定
```bash
https://api.wordpress.org/secret-key/1.1/salt/
```
生成された値を貼り付ける



### 確認方法
```bash
php -l wordpress/wp-config.php
```
OKの目安：
- No syntax errors detected と表示される

---

## 2-6. WordPress を /var/www/html に配置する


### この工程でしていること
- Apache公開領域にWordPress一式を配置する
- 作業用で編集した wp-config.php を公開側へ反映する


### 実行内容
```bash
ls -la /var/www/html
```
```bash
cp -a wordpress/. /var/www/html/
```
```bash
chown -R ec2-user:apache /var/www/html
```


### 確認方法
```bash
ls -la /var/www/html | egrep "wp-config.php|wp-admin|wp-includes"
```
OKの目安：
- wp-config.php が存在する
- wp-admin、wp-includes ディレクトリが存在する

DB設定確認
```bash
grep -n "DB_NAME\|DB_USER\|DB_PASSWORD\|DB_HOST" /var/www/html/wp-config.php
```
OKの目安：
- DB_NAME、DB_USER 等が想定通り表示される
- typo（wordpress-usr等）がない

---

## 2-7. Apache 設定変更（.htaccess 有効化）



### この工程でしていること
- WordPressのリライトルールを有効にする
- .htaccess をApacheが読み取れるようにする


### 実行内容
```bash
vi /etc/httpd/conf/httpd.conf
```

設定変更：
```bash
<Directory "/var/www/html">
    AllowOverride All
</Directory>
```


### 設定の意味

- AllowOverride All
  → .htaccess の設定を有効にする
  → パーマリンク設定が動作するようにする


### 確認方法
```bash
httpd -t
```

OKの目安：
- Syntax OK と表示される

反映（必要時）
```bash
systemctl restart httpd
```

確認：
```bash
systemctl status httpd
```
OKの目安：
- active (running) と表示される

---

## 2-8. 画像処理用 PHP モジュールを導入する


### この工程でしていること
- WordPress の画像処理機能を有効化する



### 実行コマンド
```bash
sudo dnf install php-gd
```
```bash
sudo dnf list installed | grep php-gd
```
```bash
dnf list | grep php
```
```bash
sudo dnf install -y php8.1-gd
```


### 確認方法とOKの目安
```bash
php -m | grep gd
```
- gd 表示されれば PHPに有効化されている


---

## 2-9. Apache 公開領域の権限を設定する



### この工程でしていること
- Apache が WordPress を操作できるようにする
- セキュリティと運用性を両立する


### 実行コマンド
```bash
sudo chown -R apache /var/www
```
```bash
sudo chgrp -R apache /var/www
```
```bash
sudo chmod 2775 /var/www
```
```bash
sudo find /var/www -type d -exec chmod 2775 {} \;
```
```bash
sudo find /var/www -type f -exec chmod 0644 {} \;
```


### 確認方法とOKの目安
```bash
ls -ld /var/www/html
```
- apache apache になっている

---

## 2-10. Apache を再起動する



### この工程でしていること
- 設定変更を反映する


### 実行コマンド
```bash
sudo systemctl restart httpd
```


### 確認方法とOKの目安
- エラーなく起動

---

## 2-11. 自動起動設定・最終確認



### この工程でしていること
- OS 起動時にサービスを自動起動させる
- 本番運用準備を完了する


### 実行コマンド
```bash
sudo systemctl enable httpd
```
```bash
sudo systemctl enable mariadb
```
```bash
sudo systemctl status mariadb httpd
```


### 確認方法とOKの目安
- 両方 active (running)
- enabled

---

## 2-12. Web インストール画面の確認


### この工程でしていること
- WordPress 初期設定画面を表示する


### 確認方法（ブラウザ）
`http://サーバのパブリックDNS/`

または

`http://サーバのパブリックDNS/blog/`


### OKの目安
- WordPress セットアップ画面が表示される
- サイト情報入力画面が表示される

---

## 2-13. ここまでで完了していること

- WordPress 本体導入完了
- DB 接続設定完了
- Apache 連携完了
- 権限設定完了
- 初期セットアップ準備完了

---

## 重要ポイントまとめ

- wp-config.php は最重要
- DB 権限設定は必須
- AllowOverride がないと URL が壊れる
- php-gd がないと画像不具合
- 権限ミスは最大のトラブル要因








