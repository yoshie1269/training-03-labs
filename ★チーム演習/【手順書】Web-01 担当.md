# Web-01 担当 詳細設計・構築手順書（詳細版）

担当者：[　　　　　　　]
サーバ名：web01.wp.local  
Private IP：[　　　　　　　]  
EIP：[　　　　　　　]  

---

## 1. 役割概要

本サーバ（Web-01）は、以下の役割を担当する。

- Apache によるWeb公開
- 外部DNS（BIND）による権威DNS運用
- Postfix によるSMTP送信処理

本サーバは、インターネット公開の入口となる重要サーバであるため、
EIPを付与し、Public Subnetに配置する。

---

## 2. 事前確認

作業前に以下を確認する。

- [ ] EC2インスタンス作成済み
- [ ] Public Subnetに配置済み
- [ ] EIP割当済み
- [ ] セキュリティグループ設定済み

### 必須ポート

| ポート | プロトコル | 許可元 | 用途 |
|--------|------------|--------|------|
| 80 | TCP | 0.0.0.0/0 | HTTP |
| 25 | TCP | 0.0.0.0/0 | SMTP |
| 53 | TCP/UDP | 0.0.0.0/0 | DNS |
| 22 | TCP | 管理端末IP | SSH |

- [ ] 内部DNS（wp.local）で名前解決可能
- [ ] ap01 / db01 への疎通確認済み

---

## 3. Apache 構築（名前ベースVirtualHost）

### 3-1. Apacheインストール
```bash
dnf install -y httpd
```
---

### 3-2. Apache起動設定

systemctl enable --now httpd

起動確認：
```bash
systemctl status httpd
```
active (running) であることを確認する。

---

### 3-3. 顧客用ディレクトリ作成

顧客ごとにDocumentRootを作成する。
```bash
mkdir -p /var/www/banana
mkdir -p /var/www/ringo
mkdir -p /var/www/tomato
```
権限設定：
```bash
chown -R apache:apache /var/www
chmod -R 755 /var/www
```
---

### 3-4. テストページ作成

各顧客用ディレクトリにindex.htmlを作成する。

例：

/var/www/banana/index.html  
/var/www/ringo/index.html  
/var/www/tomato/index.html

内容例：

<h1>banana site</h1>

---

### 3-5. VirtualHost設定（名前ベース）

設定ファイル作成：

/etc/httpd/conf.d/vhost.conf

記述内容：
```bash
<VirtualHost *:80>
  ServerName banana.teamX.entrycl.net
  DocumentRoot /var/www/banana

  ErrorLog logs/banana_error.log
  CustomLog logs/banana_access.log combined
</VirtualHost>

<VirtualHost *:80>
  ServerName ringo.teamX.entrycl.net
  DocumentRoot /var/www/ringo

  ErrorLog logs/ringo_error.log
  CustomLog logs/ringo_access.log combined
</VirtualHost>

<VirtualHost *:80>
  ServerName tomato.teamX.entrycl.net
  DocumentRoot /var/www/tomato

  ErrorLog logs/tomato_error.log
  CustomLog logs/tomato_access.log combined
</VirtualHost>
```
---

### 3-6. Apache設定確認・再起動

設定確認：
```bash
apachectl configtest
```
Syntax OK を確認。

再起動：
```bash
systemctl restart httpd
```
---

## 4. 外部DNS（BIND）構築

### 4-1. BINDインストール
```bash
dnf install -y bind bind-utils
```
---

### 4-2. named.conf 設定

編集：

/etc/named.conf

options 内確認：
```bash
listen-on port 53 { any; };
allow-query { any; };

zone定義追加：

zone "teamX.entrycl.net" IN {
  type master;
  file "teamX.entrycl.net.zone";
};
```
---

### 4-3. ゾーンファイル作成

作成場所：

/var/named/teamX.entrycl.net.zone

内容例：
```bash
$TTL 3600
@ IN SOA web01.wp.local. admin.teamX.entrycl.net. (
 20260101
 3600
 900
 604800
 86400
)

@       IN NS web01.wp.local.
@       IN NS web02.wp.local.

banana  IN A  <Web-01のEIP>
ringo   IN A  <Web-01のEIP>
tomato  IN A  <Web-01のEIP>
```
---

### 4-4. DNS設定チェック

構文確認：
```bash
named-checkconf
```
```bash
named-checkzone teamX.entrycl.net /var/named/teamX.entrycl.net.zone
```
---

### 4-5. named起動
```bash
systemctl enable --now named
```
確認：
```bash
systemctl status named
```
---

---

主な設定：
```bash
myhostname = web01.wp.local
mydomain = teamX.entrycl.net
myorigin = $mydomain
inet_interfaces = all
mydestination = $myhostname, localhost.$mydomain, localhost
```
---

### 5-3. 起動設定
```bash
systemctl enable --now postfix
```
確認：

systemctl status postfix

---

## 6. 動作確認

### 6-1. Web確認

ブラウザ：

http://banana.teamX.entrycl.net  
http://ringo.teamX.entrycl.net  
http://tomato.teamX.entrycl.net  

→ 各ページが表示されること

CLI：

curl http://banana.teamX.entrycl.net

---

### 6-2. DNS確認

dig banana.teamX.entrycl.net

EIPが返ることを確認。

---

### 6-3. Mail確認

mail test@test.com
subject: test
test mail
.

送信ログ確認：

tail /var/log/maillog

---

## 7. 運用・保守

### 7-1. 日常確認項目

| 項目 | コマンド |
|------|----------|
| Apache | systemctl status httpd |
| DNS | systemctl status named |
| Mail | systemctl status postfix |
| Disk | df -h |

---

### 7-2. 定期バックアップ対象

| 対象 | パス |
|------|------|
| Apache設定 | /etc/httpd |
| DNS設定 | /etc/named.conf |
| Zone | /var/named |
| Postfix | /etc/postfix |

---

## 8. 障害対応メモ

### 8-1. Web表示不可

確認：

systemctl status httpd  
journalctl -xeu httpd  
apachectl configtest

---

### 8-2. DNS名前解決不可

確認：

systemctl status named  
named-checkconf  
named-checkzone  

ログ：

/var/log/messages

---

### 8-3. Mail送信不可

確認：

systemctl status postfix  
mailq  
tail /var/log/maillog

---

## 9. 引継ぎメモ

- VirtualHost設定：/etc/httpd/conf.d/vhost.conf
- ゾーン管理：/var/named/teamX.entrycl.net.zone
- EIP変更時は必ずDNS更新する
- 設定変更後は必ずconfigtestを実施する

---
