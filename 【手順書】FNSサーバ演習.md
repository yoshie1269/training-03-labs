# NFSサーバ演習 手順書 
# 前提条件
## ① 構成

| サーバ | 役割 |
|--------|------|
| メールサーバ | Postfix / Dovecot |
| メールサーバ | Postfix / Dovecot |
| メールサーバ | Postfix / Dovecot |
| メールサーバ | Postfix / Dovecot |
| DNS + NFSサーバ | BIND / NFS |

※ 5台構成で構築する。

---

## ② 環境条件

- OS：Amazon Linux 2023
- ネットワーク：同一VPC（172.31.0.0/16）
- 権限：root / sudo 使用可

---

## ③ メールサーバ用セキュリティグループ

### ✅ 必須ポート一覧

| ポート | プロトコル | 許可元 | 用途 |
|--------|------------|--------|------|
| 25 | TCP | 0.0.0.0/0 | SMTP受信（Outlook → Postfix） |
| 110 | TCP | 172.31.0.0/16 | POP3（受信確認） |
| 143 | TCP | 172.31.0.0/16 | IMAP |
| 2049 | TCP/UDP | NFSサーバSG | NFS通信 |
| 22 | TCP | マイIP | SSH管理 |

---

## ④ NFSサーバ用セキュリティグループ

### ✅ 必須ポート一覧

| ポート | プロトコル | 許可元 | 用途 |
|--------|------------|--------|------|
| 2049 | TCP/UDP | MailサーバSG | NFS本体 |
| 111 | TCP/UDP | MailサーバSG | rpcbind |
| 22 | TCP | マイIP | SSH |

---

## ⑤ 事前確認

以下が事前に確認できていること。

- ping 疎通OK
- SSH接続可能
- DNS名前解決可能
- NFSマウント可能

---

# 1. Mail Server（Postfix）設定

---

## ① rootユーザーへ切り替え

### 実行コマンド
```bash
sudo su -
```

### 何をしているか
- 管理者権限（root）に切り替え、サーバ設定を行える状態にする。

---

## ② Postfixのインストール

### 実行コマンド
```bash
dnf install postfix
```

### 何をしているか
- メール送受信サーバ（Postfix）をインストールする。

---

## ③ Postfixの起動と動作確認

### 実行コマンド
```bash
systemctl start postfix
ps -ef | grep postfix
netstat -ln | grep 25
```

### 何をしているか
- Postfixサービスを起動する。
- プロセスが動作しているか確認する。
- メール用ポート（25番）が待受しているか確認する。

---

## ④ mailxのインストール

### 実行コマンド
```bash
dnf install mailx
dnf list installed | grep mailx
```

### 何をしているか
- メール送信テスト用コマンド（mailx）をインストールする。
- 正常にインストールされたか確認する。

---

## ⑤ Postfix設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/postfix/main.cf /etc/postfix/main.cf.date "+%Y%m%d_%H%M%S".bak
ls -l /etc/postfix/ | grep main
```

### 何をしているか
- 設定変更前の状態をバックアップとして保存する。
- バックアップファイルが作成されたか確認する。

---

## ⑥ 設定ファイルの整理（コメント削除）

### 実行コマンド
```bash
grep -v ^# /etc/postfix/main.cf | cat -s > /tmp/main.cf
cp /tmp/main.cf /etc/postfix/main.cf
```

### 何をしているか
- コメント行を除去し、設定内容を見やすくする。
- 整理した内容を正式な設定ファイルに反映する。

---

## ⑦ Postfix設定の編集

### 実行コマンド
```bash
vi /etc/postfix/main.cf
```

### 追記内容
```bash
myhostname = mail-onishi2.teamg.entrycl.net
mydomain = teamg.entrycl.net
myorigin = $myhostname
mynetworks = 172.31.0.0/16, 127.0.0.1
mail_spool_directory = /var/spool/mail/
```

### 変更内容
```bash
inet_interfaces = all
mydestination = $mydomain, $myhostname
```

### 何をしているか
- メールサーバのホスト名・ドメインを設定する。
- メール送信を許可するネットワーク範囲を指定する。
- メール保存場所を設定する。
- 外部通信を受け付けるように設定する。
- 自サーバ宛メールの受信設定を行う。

---

## ⑧ Postfixの再起動

### 実行コマンド
```bash
systemctl restart postfix
```

### 何をしているか
- 設定変更を反映させるため、Postfixを再起動する。

---

## ⑨ メール送信テスト

### 実行コマンド
```bash
mail -s onishi2 root@mail-onishi2.teamg.entrycl.net
mail
```

### 何をしているか
- 実際にメールを送信して動作確認を行う。
- メールボックスを確認する。

---

## ⑩ ログ管理サービス（rsyslog）の導入

### 実行コマンド
```bash
dnf install rsyslog
dnf list installed | grep rsyslog
systemctl start rsyslog
systemctl status rsyslog
```

### 何をしているか
- ログ管理サービスをインストールする。
- 起動して正常に動作しているか確認する。

---

## ⑪ メールログの確認

### 実行コマンド
```bash
less /var/log/maillog
```

### 何をしているか
- メールの送受信履歴やエラー内容を確認する。
- トラブル時の原因調査に使用する。

---
---

# 2. DNS + NFSサーバ構築

## ①BIND（DNSサーバ）の構築
## ①-1. BINDのインストール

### 実行コマンド
```bash
dnf install bind
```

### 何をしているか
- DNSサーバソフト（BIND）をインストールする。

---

## ①-2. BIND設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/named.conf /etc/named.conf.date "+%Y%m%d_%H%M%S".bak
```

### 何をしているか
- 設定変更前の状態をバックアップとして保存する。

---

## ①-3. BIND設定ファイルの編集

### 実行コマンド
```bash
vi /etc/named.conf
```

### 設定内容
```bash
- listen設定の変更（コメントアウト）
#listen-on port 53 { 127.0.0.1; };
#listen-on-v6 port 53 { ::1; };

- クエリ許可設定
allow-query { any; };

- ゾーン設定の追加（末尾に追記）
zone "teamg.entrycl.net" IN {
type master;
file "/var/named/teamg.entrycl.net.zone";
};
```

### 何をしているか
- DNSを外部から利用できるように設定する。
- 管理するドメイン（ゾーン）を定義する。
- ゾーンファイルの場所を指定する。

---

## ①-4. BIND設定ファイルの構文チェック

### 実行コマンド
```bash
named-checkconf
```
### 何をしているか
- 設定ファイルに文法エラーがないか確認する。

---

## ①-5. ゾーンファイルの作成

### 実行コマンド
```bash
vi /var/named/teamg.entrycl.net.zone
```
### 設定内容
```bash
$TTL 3600
@ IN SOA ns.teamg.entrycl.net. test.teamg.entrycl.net. (
20210401
3600
3600
3600
3600 )
IN NS ns.teamg.entrycl.net.
ns IN A 52.39.185.90

@ IN MX 10 mail-onishi.teamg.entrycl.net.
@ IN MX 10 mail-koinuma.teamg.entrycl.net.
@ IN MX 10 mail-sakamoto.teamg.entrycl.net.
@ IN MX 10 mail-furuya.teamg.entrycl.net.

mail-onishi IN A 16.148.201.121
mail-koinuma IN A 34.219.111.156
mail-sakamoto IN A 35.86.75.167
mail-furuya IN A 34.211.175.39
```

### 何をしているか
- ドメインのDNS情報を定義する。
- ネームサーバ、メールサーバ、IPアドレスを登録する。
- MXレコードでメール配送先の優先順位を設定する。

---

## ①-6. ゾーンファイルの構文チェック

### 実行コマンド
```bash
named-checkzone teamg.entrycl.net /var/named/teamg.entrycl.net.zone
```

### 何をしているか
- ゾーンファイルに記述ミスがないか確認する。

---

## ①-7. namedサービスの起動

### 実行コマンド
```bash
systemctl start named
systemctl status named
```

### 何をしているか
- DNSサーバを起動する。
- 正常に動作しているか確認する。

---

## ①-8. DNS動作確認

### 実行コマンド
ローカルDNS確認
```bash
dig @[DNSサーバIP] teamg.entrycl.net MX
```

外部向けDNS確認
```bash
dig teamg.entrycl.net MX
```

### 何をしているか
- DNSサーバが正しく名前解決できるか確認する。
- MXレコードが取得できるか確認する。

---

## ② NFSサーバの構築
## ②-1. 共有ディレクトリの作成

### 実行コマンド
```bash
mkdir /share_teamg
```

### 何をしているか
- クライアントに共有するディレクトリを作成する。

---

## ②-2. exports設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/exports /etc/exports.`date "+%Y%m%d_%H%M%S"`.bak
```

### 何をしているか
- NFS設定ファイルのバックアップを作成する。

---

## ②-3. exports設定の編集

### 実行コマンド
```bash
vi /etc/exports
```

### 設定内容
```bash
/share_teamg 172.31.0.0/16(rw,sync,no_root_squash)
```

### 何をしているか
- 共有ディレクトリの公開範囲を設定する。
- 読み書き可能なネットワークを指定する。
- root権限の制限を解除する。

---

## ②-4. NFS設定の反映

### 実行コマンド
```bash
systemctl restart nfs-server
systemctl enable nfs-server
exportfs -a
```

### 何をしているか
- NFSサービスを再起動する。
- 自動起動を有効化する。
- exports設定を反映する。






# 3. Dovecot（メール受信サーバ）設定

## ①メール受信用ユーザーの作成

---

## ①-1. ユーザー作成

### 実行コマンド
```bash
useradd yoshie -g mail -M -K MAIL_DIR=/dev/null -s /sbin/nologin
passwd yoshie
```

### 何をしているか
- メール専用ユーザーを作成する。
- ログインできない設定にする。
- パスワードを設定する。

---

## ② Dovecotのインストール
## ②-1. Dovecotの導入

### 実行コマンド
```bash
dnf install dovecot
dnf list installed | grep dovecot
```

### 何をしているか
- メール受信サーバ（Dovecot）をインストールする。
- 正しく導入されたか確認する。

---

## ③ Dovecot基本設定
## ③-1. 設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/dovecot/dovecot.conf /etc/dovecot/dovecot.conf.bak
```

### 何をしているか
- 設定変更前の状態を保存する。

---

## ③-2. dovecot.confの編集

### 実行コマンド
```bash
vi /etc/dovecot/dovecot.conf
```

### 設定内容
```bash
protocols = pop3
mail_location = maildir:/var/spool/mail/%u/
```

### 何をしているか
- POP3通信を有効化する。
- メール保存先ディレクトリを指定する。

---

## ④ SSL設定の無効化
## ④-1. SSL設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/dovecot/conf.d/10-ssl.conf /etc/dovecot/conf.d/10-ssl.conf.date "+%Y%m%d_%H%M%S".bak
ls -l /etc/dovecot/conf.d/ | grep 10-ssl
```

### 何をしているか
- SSL設定ファイルのバックアップを作成する。

---

## ④-2. SSL設定の編集

### 実行コマンド
```bash
vi /etc/dovecot/conf.d/10-ssl.conf
```

### 設定内容
`（ssl関連行をコメントアウト）`

### 何をしているか
- SSL通信を無効化する。
- 平文通信を利用できるようにする。

---

## ⑤ 認証設定の変更
## ⑤-1. 認証設定ファイルのバックアップ

### 実行コマンド
```bash
cp /etc/dovecot/conf.d/10-auth.conf /etc/dovecot/conf.d/10-auth.conf.date "+%Y%m%d_%H%M%S".bak
ls -l /etc/dovecot/conf.d/ | grep 10-auth
```

### 何をしているか
- 認証設定ファイルのバックアップを作成する。

---

## ⑤-2. 認証設定の編集

### 実行コマンド
```bash
vi /etc/dovecot/conf.d/10-auth.conf
```

### 設定内容
`disable_plaintext_auth = no`

### 何をしているか
- 平文認証を許可する。
- メールクライアントからの接続を可能にする。

---

## ⑥ Dovecotサービスの起動
## ⑥-1. サービス起動・自動起動設定

### 実行コマンド
```bash
systemctl start dovecot
systemctl enable dovecot
```

### 何をしているか
- Dovecotを起動する。
- OS起動時に自動起動するよう設定する。

---

## ⑦ NFSによるメール領域マウント
## ⑦-1. fstab設定

### 実行コマンド
```bash
vi /etc/fstab
```

### 追記内容
`NFSサーバIP:/magic /var/spool/mail nfs4 defaults 0 0`

### 何をしているか
- NFS上の共有領域をメール保存用として使用する。
- 再起動後も自動マウントされるよう設定する。

---

## ⑦-2. マウント実行・確認

### 実行コマンド
```bash
mount /var/spool/mail
df
```

### 何をしているか
- 設定を反映してマウントする。
- 正しく接続されているか確認する。

---

## ⑧ メール保存領域の権限設定
## ⑧-1. 権限確認

### 実行コマンド
```bash
ls -ld /var/spool/mail
```

### 何をしているか
- 現在のディレクトリ権限を確認する。

---

## ⑧-2. 権限変更

### 実行コマンド
```bash
chown -R root:mail /var/spool/mail
chmod -R 770 /var/spool/mail
```

### 何をしているか
- メール用グループに権限を付与する。
- Outlookなどからの利用を可能にする。

---

## ⑨ 各サービスの再起動
## ⑨-1. Postfix・Dovecot再起動

### 実行コマンド
```bash
systemctl restart postfix
systemctl restart dovecot
```

### 何をしているか
- 設定変更を反映させるため再起動する。

---

## ⑩ 動作確認
## ⑩-1. ログ監視

### 実行コマンド
```bash
tail -f /var/log/maillog
```

### 何をしているか
- メール処理状況をリアルタイムで確認する。

---

## ⑩-2. メール確認

### 実行コマンド
```bash
mail
ls /var/spool/mail/ユーザ名/new
```

### 何をしているか
- メールが正常に届いているか確認する。
- 新着メール保存先を確認する。

---