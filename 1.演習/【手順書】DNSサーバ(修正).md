# DNS総合演習 手順書（teamg.entrycl.net 構成）

---

# 1. 概要

本手順書では、以下の構成を構築する。

- EC2-1：権威DNS（BIND）＋DB
- EC2-2：Web（WordPress）
- ドメイン：teamg.entrycl.net
- WordPress から DB へホスト名（db.teamg.entrycl.net）で接続

DNS → Web → DB をすべてホスト名で連携させる。

---

# 2. 構成図
```bash

                [ Internet ]
                      |
                      v
          entrycl.net（親ドメイン）
                      |
                  NS委譲
                      |
                      v
    +----------------------------------+
    | EC2-1（DNS + DB）                |
    |----------------------------------|
    | BIND（権威DNS）                  |
    | MariaDB                          |
    | Public IP: 52.41.166.54          |
    | Private IP: 172.31.22.149        |
    +----------------------------------+
                      |
                 同一VPC通信
                      |
    +----------------------------------+
    | EC2-2（Web）                     |
    |----------------------------------|
    | Apache / PHP / WordPress         |
    | Public IP: 52.88.132.64          |
    +----------------------------------+
```

---

# 3. 使用サーバ

| 役割 | サーバ | IP |
|------|--------|----|
| DNS + DB | EC2-1 | 52.41.166.54 / 172.31.22.149 |
| Web | EC2-2 | 52.88.132.64 |

---

# 【構成①】権威DNS構築（EC2-1）

---

## 1. BIND インストール

### この工程でしていること
- 権威DNSサーバをインストールする

### 実行コマンド（EC2-1）
```bash
dnf install bind -y
```
### コマンドの意味
- bind：DNSサーバ本体
- -y：確認なし実行

### 確認方法
```bash
dnf list installed | grep bind
```
### OKの目安
- bind が表示される

---

## 2. named 起動
```bash
systemctl start named
```
```bash
systemctl enable named
```

### この工程でしていること
- DNSサービスを起動
- OS再起動後も自動起動

### 確認
```bash
systemctl status named
```
OKの目安：
- active (running)

---

## 3. named.conf 設定
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
zone “teamg.entrycl.net” IN {
type master;
file “/var/named/teamg.entrycl.net.zone”;
};
```
---

### 各設定の意味

| 設定 | 意味 |
|------|------|
| listen-on | 外部からの53番アクセス許可 |
| allow-query any | 全ネットワークから問い合わせ許可 |
| zone | 管理するドメイン定義 |
| type master | このサーバが権威DNS |

---

### 設定確認
```bash
named-checkconf
```
OK：
- エラーなし

---

# 4. ゾーンファイル作成（最重要）
```bash
vi /var/named/teamg.entrycl.net.zone
```
---

## 設定内容（指定内容を反映）
```bash
$TTL 30
 IN SOA ns.teamg.entrycl.net. admin.teamg.entrycl.net. (
20260225 ; serial
3600 ; refresh
3600 ; retry
3600 ; expire
3600 ) ; minimum

IN NS ns.teamg.entrycl.net.

ns  IN A 52.41.166.54
web IN A 52.88.132.64
db  IN A 172.31.22.149
```
---

# ゾーンファイルの詳細解説

---

## $TTL 30

意味：
- DNSキャッシュ有効時間（30秒）
- 設定変更の反映を早くするため短く設定

---

## SOA レコード
```bash
 IN SOA ns.teamg.entrycl.net. admin.teamg.entrycl.net.
```
| 項目 | 意味 |
|------|------|
| @ | zoneのルート |
| SOA | 権威情報 |
| ns.teamg... | 主DNS |
| admin... | 管理者メール（@を.に置換） |

---

## serial（20260225）

意味：
- ゾーン更新番号
- 修正時は必ず増やす

---

## NS レコード
```bash
IN NS ns.teamg.entrycl.net.
```
意味：
- このドメインのDNSは ns.teamg... であると宣言

---

# Aレコードの詳細説明（重要）

---

## ns IN A 52.41.166.54

意味：
- ns.teamg.entrycl.net → 52.41.166.54
- DNSサーバ自身のIP（Public）

用途：
- 親ドメインからの委譲先

---

## web IN A 52.88.132.64

意味：
- web.teamg.entrycl.net → WebサーバPublic IP

用途：
- インターネット公開用
- ブラウザアクセス用

---

## db IN A 172.31.22.149

意味：
- db.teamg.entrycl.net → DBサーバPrivate IP

用途：
- VPC内部通信用
- WordPressからの接続専用

重要：
- 外部公開しないためPrivate IP

---

# 5. ゾーンチェック
```bash
named-checkzone teamg.entrycl.net /var/named/teamg.entrycl.net.zone
```

OK：
- OK と表示

---

# 6. named 再起動
```bash
systemctl restart named
```
---

# 7. 名前解決確認（EC2-2）
```bash
dig @52.41.166.54 web.teamg.entrycl.net
```
```bash
dig @52.41.166.54 db.teamg.entrycl.net
```
OKの目安：

web → 52.88.132.64  
db → 172.31.22.149

---

# WordPress 側 DB接続確認
```bash
mysql -u wpuser -h db.teamg.entrycl.net -p
```
OK：
- 接続成功

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

# 最終チェックリスト

- named 起動
- named-checkzone OK
- dig 成功
- web表示成功
- DBホスト名接続成功
- WordPress稼働

---

# 完了状態

- teamg.entrycl.net が権威DNSで管理されている
- web.teamg.entrycl.net でWeb公開
- db.teamg.entrycl.net で内部DB接続
- DNS → Web → DB がホスト名連携
