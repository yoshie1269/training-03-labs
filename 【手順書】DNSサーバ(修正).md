# DNS総合演習 手順書（権威DNS＋名前解決＋WordPress＋DBホスト名接続）

---

# 1. 概要

本演習では、以下の統合構成を構築する。

1. EC2 2台構成で権威DNSを構築する
2. 別EC2から名前解決を行う
3. WordPress をドメインで公開する
4. WordPress から DB へ「ホスト名」で接続する

DNS → Web → DB の連携構成を完成させる。

---

# 2. 構成図
```bash

            [ Internet ]
                  |
                  v
        entry.net（レジストラ）
                  |
              NS委譲
                  |
                  v
  +----------------------------------+
  |          EC2-1（DNS + DB）        |
  |----------------------------------|
  | BIND（権威DNS）                   |
  | MariaDB（DBサーバ）               |
  | IP: 172.31.10.10                 |
  +----------------------------------+
                  |
             同一VPC通信
                  |
  +----------------------------------+
  |          EC2-2（Web）             |
  |----------------------------------|
  | Apache                           |
  | PHP                              |
  | WordPress                        |
  | IP: 172.31.10.20                 |
  +----------------------------------+

```

---

# 3. 使用サーバ

| 役割 | EC2 | 主な機能 |
|------|------|----------|
| DNS + DB | EC2-1 | BIND / MariaDB |
| Web | EC2-2 | Apache / PHP / WordPress |

---

# 4. 前提条件

- entry.net にてドメイン取得済み
- EC2 は同一 VPC 内
- Public IP 割当済み
- セキュリティグループ設定済み
  - 53/TCP・UDP
  - 80/TCP
  - 3306/TCP（Web→DBのみ）

---

# 【構成①】権威DNS構築と名前解決

---

## 1. BIND インストール（EC2-1）

### この工程でしていること
- 権威DNSサーバを構築する

### 実行コマンド（EC2-1）
```bash
dnf install bind -y
```

### 確認方法
```bash
dnf list installed | grep bind
```

### OKの目安
- bind パッケージが表示される

---

## 2. named 起動（EC2-1）

### この工程でしていること
- DNSサービスを起動し、自動起動設定する

### 実行コマンド
```bash
systemctl start named
```
```bash
systemctl enable named
```

### 確認方法
```bash
systemctl status named
```
### OKの目安
- active (running)

---

## 3. named.conf 設定（EC2-1）

### この工程でしていること
- 外部からの問い合わせを許可する
- ゾーンを登録する

### 実行コマンド
```bash
vi /etc/named.conf
```
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
zone “example.entry.net” IN {
type master;
file “/var/named/example.entry.net.zone”;
};
```

### 確認方法
```bash
named-checkconf
```
### OKの目安
- エラーが出ない

---

## 4. ゾーンファイル作成（EC2-1）

### この工程でしていること
- Aレコードを登録する
- NSレコードを設定する

### 実行コマンド
```bash
vi /var/named/example.entry.net.zone
```
### 記載内容
```bash
$TTL 3600
@ IN SOA ns.example.entry.net. admin.example.entry.net. (
20220401
3600
3600
3600
3600 )

IN NS ns.example.entry.net.

ns  IN A 172.31.10.10
web IN A 172.31.10.20
db  IN A 172.31.10.10
```

### 確認方法
```bash
named-checkzone example.entry.net /var/named/example.entry.net.zone
```

### OKの目安
- OK と表示される

---

## 5. 名前解決確認（EC2-2）

### この工程でしていること
- クライアントからDNS問い合わせを行う

### 実行コマンド（EC2-2）
```bash
dig @172.31.10.10 web.example.entry.net
```

### OKの目安
- 172.31.10.20 が返る

---

# 【構成②】WordPress 公開（DNS連携）

---

## 6. LAMP 構築（EC2-2）

### この工程でしていること
- Web公開環境を構築する

### 実行コマンド
```bash
dnf install httpd php php-mysqlnd php-fpm wget -y
```
```bash
systemctl start httpd
```
```bash
systemctl enable httpd
```

### 確認方法
```bash
systemctl status httpd
```

### OKの目安
- active (running)

---

## 7. WordPress 導入（EC2-2）

### この工程でしていること
- WordPress をWeb公開領域へ配置する

### 実行コマンド
```bash
wget https://wordpress.org/latest.tar.gz
```
```bash
tar -xzf latest.tar.gz
```
```bash
cp -r wordpress/* /var/www/html/
```

### 確認方法
```bash
ls /var/www/html
```
### OKの目安
- wp-config-sample.php が存在

---

## 8. MariaDB 構築（EC2-1）

### この工程でしていること
- DBサーバを構築する

### 実行コマンド
```bash
dnf install mariadb105-server -y
```
```bash
systemctl start mariadb
```
```bash
systemctl enable mariadb
```

### OKの目安
- active (running)

---

## 9. WordPress 用 DB 作成（EC2-1）

### この工程でしていること
- WordPress 専用DBとユーザーを作成する

### 実行コマンド
```bash
mysql -u root -p
```

```bash
CREATE DATABASE wpdb;
CREATE USER ‘wpuser’@’%’ IDENTIFIED BY ‘password’;
GRANT ALL ON wpdb.* TO ‘wpuser’@’%’;
FLUSH PRIVILEGES;
```

### 確認方法
```bash
SHOW DATABASES;
```

### OKの目安
- wpdb が存在

---

## 10. wp-config 設定（EC2-2）

### この工程でしていること
- DBへホスト名で接続する

### 実行コマンド
```bash
vi /var/www/html/wp-config.php
```
### 設定内容
```bash
define(‘DB_NAME’,‘wpdb’);
define(‘DB_USER’,‘wpuser’);
define(‘DB_PASSWORD’,‘password’);
define(‘DB_HOST’,‘db.example.entry.net’);
```

### 意味
- DB_HOST にIPではなくホスト名を指定
- DNS経由接続を実現

---

## 11. DB接続確認（EC2-2）

### この工程でしていること
- ホスト名解決＋DB接続確認

### 実行コマンド
```bash
mysql -u wpuser -h db.example.entry.net -p
```

### OKの目安
- 接続成功

---

## 12. Web公開確認

ブラウザでアクセス
```bash
http://web.example.entry.net/
```

### OKの目安
- WordPress 初期画面表示

---

# セキュリティ設計

---

## DNS/DB側 SG

| Port | Source |
|------|--------|
| 53 | 0.0.0.0/0 |
| 3306 | Web-SG |
| 22 | 自分IP |

---

## Web側 SG

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
- named未起動
- NS委譲ミス
- SG53未開放

---

## DB接続エラー

原因：
- DB_HOST IP指定
- DNS未反映
- 3306未開放
- GRANT '%' 未設定

---

## Web表示不可

原因：
- SG80未開放
- Apache未起動
- DocumentRootミス

---

# 最終確認チェックリスト

- dig 成功
- ns レコード解決
- web レコード解決
- DBホスト名接続成功
- WordPress 表示成功

---

# 完了状態

- 権威DNS稼働
- ドメインでWeb公開
- DBホスト名接続成功
- WordPress 正常稼働


⸻

