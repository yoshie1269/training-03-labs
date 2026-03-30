# DNS総合演習 手順書（権威DNS＋名前解決＋WordPress連携）

---

## 概要

本演習では、以下の構成を構築する。

1. EC2 2台構成で権威DNSを構築し、別EC2から名前解決を行う
2. 権威DNS＋WordPress を構築し、ドメインでWeb公開する
3. WordPress から DB へホスト名で接続する

---

## 構成概要
```bash
[Internet]
|
v
[entry.net（レジストラ）]
|
| NS委譲
v
[EC2-DNS/DB] --------- DNS --------> [EC2-WEB]
BIND / MariaDB Apache / PHP / WP
権威DNS クライアント
```
---

## 使用サーバ

| 役割 | EC2 |
|------|------|
| DNS + DB | EC2-1 |
| Web | EC2-2 |

---

## 前提条件

- entry.net でサブドメインを取得済み
- EC2 は同一 VPC に配置
- Public IP 割当済み
- 53 / 80 / 3306 開放済み

---

# 【構成①】権威DNS構築と名前解決

---

## 1. BIND インストール（EC2-1）

### この工程でしていること
- 権威DNSサーバ構築

### 実行コマンド
```bash
dnf install bind -y
```
---

## 2. named 起動
```bash
systemctl start named
```
```bash
systemctl enable named
```
---

## 3. named.conf バックアップ
```bash
cp /etc/named.conf /etc/named.conf.bak
```
---

## 4. named.conf 編集
```bash
vi /etc/named.conf
```
---

### 修正内容
コメントアウト
```bash
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };
```
変更
```bash
allow-query { any; };
```
追記
```bash
zone "example.entry.net" IN {
type master;
file "/var/named/example.entry.net.zone";
};
```

---

## 5. 設定チェック
```bash
named-checkconf
```
---

## 6. ゾーンファイル作成
```bash
vi /var/named/example.entry.net.zone
```
---

### 記載内容
```bash
$TTL 3600
@ IN SOA ns.example.entry.net. admin.example.entry.net. (
20220401 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum

 IN NS ns.example.entry.net.
ns IN A 172.31.10.10
web IN A 172.31.10.20
db IN A 172.31.10.10
```

---

## 7. ゾーンチェック
```bash
named-checkzone example.entry.net /var/named/example.entry.net.zone
```
---

## 8. named 再起動
```bash
systemctl restart named
```
---

## 9. クライアント側名前解決（EC2-2）
```bash
dig @172.31.10.10 web.example.entry.net
```
---

### OKの目安
- Aレコードが返る

---

# 【構成②】WordPress 公開（DNS連携）

---

## 10. entry.net 側 NS 委譲設定

### この工程でしていること
- 公開DNS委譲

---

## 11. LAMP構築（EC2-2）
```bash
dnf install httpd php php-mysqlnd php-fpm wget -y
```
```bash
dnf install mariadb-client -y
```
---

## 12. Apache 起動
```bash
systemctl start httpd
```
```bash
systemctl enable httpd
```
---

## 13. WordPress 導入
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```
```bash
cp -r wordpress/* /var/www/html/
```
---

## 14. DB構築（EC2-1）
```bash
dnf install mariadb105-server -y
```
```bash
systemctl start mariadb
```
```bash
systemctl enable mariadb
```
---

## 15. DB初期設定
```bash
mysql_secure_installation
```
---

## 16. WordPress用DB作成
```bash
mysql -u root -p
```
```bash
CREATE DATABASE wpdb;
CREATE USER 'wpuser'@'%' IDENTIFIED BY 'password';

GRANT ALL ON wpdb.* TO 'wpuser'@'%';

FLUSH PRIVILEGES;
```

---

## 17. wp-config 設定（EC2-2）
```bash
vi /var/www/html/wp-config.php
```
---

### 修正内容
```bash
define('DB_NAME','wpdb');
define('DB_USER','wpuser');
define('DB_PASSWORD','password');
define('DB_HOST','db.example.entry.net');
```
---

## 18. Apache 設定
```bash
vi /etc/httpd/conf/httpd.conf
<Directory "/var/www/html">
AllowOverride All
</Directory>
```
---

## 19. Apache 再起動
```bash
systemctl restart httpd
```
---

## 20. DB接続確認（ホスト名）
```bash
mysql -u wpuser -h db.example.entry.net -p
```
---

### OK目安
- 接続成功

---

## 21. WordPress 初期設定

ブラウザ

http://web.example.entry.net/

---

## 22. DNS最終確認
```bash
dig web.example.entry.net
```
```bash
dig db.example.entry.net
```
---

# セキュリティ設計（重要）

---

## セキュリティグループ

### DNS/DB側

| Port | Source |
|------|--------|
| 53 | 0.0.0.0/0 |
| 3306 | Web-SG |
| 22 | 自分IP |

---

### Web側

| Port | Source |
|------|--------|
| 80 | 0.0.0.0/0 |
| 22 | 自分IP |

---

# トラブル対策（試験頻出）

---

## 名前解決不可

原因：
- allow-query 未設定
- listen-on 制限
- NS委譲ミス

---

## WordPress DB接続エラー

原因：
- DB_HOST IP指定
- DNS未反映
- 3306未開放
- bind-address未設定

---

## 公開できない

原因：
- SG80閉鎖
- Apache停止
- DocumentRootミス

---

# まとめ

- 権威DNS＝BIND
- NS委譲＝公開必須
- DB_HOST＝ホスト名
- DNS→Web→DB連携
- SG×BIND×Apache

---

## 完了状態

- dig成功
- ドメイン表示
- DBホスト名接続
- WordPress稼働