# Zabbix 監視対象登録・アラート通知構築 手順書（Agent + Mail連携）

---

## 概要

本手順書では、以下の構成を構築する。

- 監視対象サーバに Zabbix Agent を導入
- Zabbix Server との通信確認
- Web 画面でのホスト登録
- CPU 監視設定
- 負荷テストによる動作検証
- Postfix を利用したアラートメール送信設定

---

## 構成

| 役割 | IP |
|------|----|
| Zabbix Server | 172.31.30.185 |
| Zabbix Agent | 172.31.18.194 |
| SMTP | Server側 Postfix |

通知経路：
```bash
Zabbix → localhost:25 → Postfix → 管理者メール
```
---

# ■ ① 監視対象サーバ（Agent側）設定

---

## 1. タイムゾーン設定

---

### この工程でしていること
- ログ時刻を日本時間に統一する

---

### 実行コマンド
```bash
timedatectl set-timezone Asia/Tokyo
```
---

### 確認方法
```bash
timedatectl
```
- Asia/Tokyo が表示される

---

## 2. Zabbix リポジトリ追加

---

### この工程でしていること
- Zabbix 公式パッケージを利用可能にする

---

### 実行コマンド
```bash
rpm -Uvh https://repo.zabbix.com/zabbix/7.4/release/amazonlinux/2023/noarch/zabbix-release-latest-7.4.amzn2023.noarch.rpm
```
```bash
dnf clean all
```
---

### 確認方法
```bash
dnf repolist | grep zabbix
```
---

## 3. Zabbix Agent インストール

---

### この工程でしていること
- 監視対象用エージェントを導入する

---

### 実行コマンド
```bash
dnf install -y zabbix-agent
```
---

### 確認方法
```bash
dnf list installed | grep zabbix-agent
```
---

## 4. Agent 設定編集

---

### この工程でしていること
- Zabbix Server と通信できるよう設定する

---

### 実行コマンド
```bash
vi /etc/zabbix/zabbix_agentd.conf
```
---

### 設定内容
```bash
Server=172.31.30.185
ServerActive=172.31.30.185
Hostname=onishi_zabbix
ListenPort=10050
ListenIP=0.0.0.0
StartAgents=5
```
---

### 各項目の意味

| 項目 | 内容 |
|------|------|
| Server | 許可サーバ |
| ServerActive | Active監視先 |
| Hostname | 登録名 |
| ListenPort | 受信ポート |
| ListenIP | 待受IP |
| StartAgents | プロセス数 |

---

## 5. Agent 起動・自動起動

---

### 実行コマンド
```bash
systemctl enable zabbix-agent
```
```bash
systemctl start zabbix-agent
```
---

### 確認方法
```bash
systemctl status zabbix-agent
```
- active (running)

---

# ■ ② Agent 動作確認

---

## 1. 待ち受け確認

---

### 実行コマンド
```bash
ss -lntp | grep 10050
```
---

### OK目安

LISTEN 0.0.0.0:10050

---

## 2. Collector 確認

---

### 実行コマンド
```bash
ps -ef | grep zabbix_agentd | grep collector
```
---

### OK目安
- collector プロセスが存在

---

# ■ ③ AWS セキュリティグループ設定

---

## この工程でしていること
- Server から Agent への通信を許可する

---

### 設定内容

Inbound Rule：

| 項目 | 値 |
|------|----|
| Type | TCP |
| Port | 10050 |
| Source | 172.31.30.185/32 |

---

### 確認方法
- AWS コンソールで設定確認

---

# ■ ④ Zabbix Server 側 通信確認

---

## この工程でしていること
- Server → Agent 通信確認

---

### 実行コマンド
```bash
zabbix_get -s 172.31.18.194 -k system.cpu.util[,user]
```
---

### OK目安

0.008335（数値が返る）

---

# ■ ⑤ Zabbix Web ホスト登録

---

## 設定手順

設定 → ホスト → 作成

---

### 入力内容

| 項目 | 値 |
|------|----|
| 名前 | onishi_zabbix |
| グループ | Zabbix servers |
| IP | 172.31.18.194 |
| Port | 10050 |

---

### OK目安
- エラーなく保存

---

# ■ ⑥ CPU 監視アイテム作成

---

## 設定手順

設定 → ホスト → onishi_zabbix → アイテム → 作成

---

### 設定内容

| 項目 | 値 |
|------|----|
| 名前 | CPU使用率 |
| タイプ | Zabbixエージェント |
| キー | system.cpu.util[,user] |
| 型 | 浮動小数 |
| 単位 | % |
| 間隔 | 60s |

---

### OK目安
- データ取得開始

---

# ■ ⑦ Web 画面確認

---

## 確認手順

監視 → 最新データ → onishi_zabbix

---

### OK目安
- CPU値が表示

---

# ■ ⑧ CPU 負荷テスト

---

## この工程でしていること
- 監視の反応確認

---

### 実行コマンド
```bash
dnf install -y stress
```
```bash
stress –cpu 2 –timeout 300
```
---

### OK目安
- CPU 使用率上昇

---

# ■ ⑨ 負荷中データ確認

---

## 確認画面

監視 → 最新データ
監視 → グラフ

---

### OK目安
- グラフ上昇

---

# ■ ⑩ テスト終了確認

---

### 実行コマンド（強制終了）
```bash
pkill stress
```
---

### OK目安
- CPU値低下

---

# ■ アラートメール設定（Postfix連携）

---

# ① Postfix 導入

---

### 実行コマンド
```bash
dnf install -y postfix mailx
```
```bash
systemctl enable –now postfix
```
---

### 確認
```bash
ss -lntp | grep :25
```
- LISTEN 127.0.0.1:25

---

# ② SMTP 疎通確認

---

### 実行コマンド
```bash
bash -lc ‘exec 3<>/dev/tcp/127.0.0.1/25; head -n 1 <&3’
```
---

### OK目安

220 ESMTP Postfix

---

# ③ メディアタイプ設定

---

## 設定場所

管理 → メディアタイプ → Email

---

### 設定内容

| 項目 | 値 |
|------|----|
| SMTP | 127.0.0.1 |
| Port | 25 |
| Security | None |
| Auth | None |
| From | zabbix@localhost |

---

# ④ Admin メール登録

---

管理 → ユーザー → Admin → メディア

---

- Email 追加
- 宛先登録

---

# ⑤ アクション設定

---

設定 → アクション → トリガーアクション

---

- Admin
- Email
- メッセージ送信

---

# ⑥ CPU トリガー設定

---

### 設定例
```bash
avg(/onishi_zabbix/system.cpu.util,1m)>80
```
---

### 深刻度

High

---

# ⑦ アラート発火テスト

---
```bash
stress –cpu 2 –timeout 300
```
---

# ⑧ アラート確認

---

## Web

監視 → 問題

---

## Mail
- 受信確認

---

# ⑨ ログ確認

---

### Zabbix
```bash
tail -n 100 /var/log/zabbix/zabbix_server.log
```
---

### Postfix
```bash
tail -n 100 /var/log/maillog
```
---

# ⑩ よくあるトラブル

---

| 症状 | 原因 |
|------|------|
| No media | Email未設定 |
| SMTP NG | Postfix未起動 |
| 通信不可 | SG未設定 |
| CPU低 | stress不足 |

---

# ■ 完成状態

---

✔ Agent 登録  
✔ データ取得  
✔ アラート発火  
✔ メール通知  
✔ 復旧通知  

＝ 運用監視環境 完成