# NTP構築手順（chrony）

本手順では以下を構築する。

- NTPサーバ
- NTPクライアント
- AWS内部NTPサーバと同期
- クライアントへの時刻配布

---

# 0 セキュリティグループ設計

## この工程でしていること

NTP通信に必要なポートを許可する。

NTPは **UDP 123ポート** を使用する。


## NTPサーバ側 セキュリティグループ

### インバウンドルール

|タイプ|プロトコル|ポート|送信元|
|---|---|---|---|
NTP|UDP|123|172.31.0.0/16|

### 意味

- UDP 123 = NTP通信
- クライアントからNTPサーバへ時刻問い合わせを許可

---

# 1 NTPサーバ設定

## 1-1 タイムゾーン変更

### 現在確認
```bash
timedatectl
```


### タイムゾーン変更
```bash
timedatectl set-timezone Asia/Tokyo
```

### 変更確認
```bash
timedatectl
```

### 確認例
```bash
Time zone: Asia/Tokyo (JST)
```


## 1-2 chronyインストール

### この工程でしていること

chronyは Linux のNTPサービス。

NTPサーバとして時刻同期を行う。


### インストール
```bash
dnf install -y chrony
```


### インストール確認
```bash
rpm -qa | grep chrony
```


## 1-3 chrony設定ファイル編集

### バックアップ作成
```bash
cp -a /etc/chrony.conf /etc/chrony.conf.bak_$(date +%Y%m%d_%H%M%S)
```


### 確認
```bash
ls /etc/ | grep chrony
```


### 編集
```bash
vi /etc/chrony.conf
```


### 設定変更

#### コメントアウトを外してIP範囲変更
```bash
allow 172.31.0.0/16
```
意味

クライアントからのNTPアクセス許可


#### コメントアウトを外す
```bash
local stratum 10
```
意味

外部NTPに接続できない場合  
自サーバを時刻源として動作させる

#### 追記
```bash
server 169.254.169.123 prefer iburst
```

意味

AWS内部NTPサーバを利用

|オプション|意味|
|---|---|
prefer|優先同期サーバ|
iburst|起動時高速同期|



## 1-4 chrony起動

### 起動
```bash
systemctl start chronyd
```


### 自動起動設定
```bash
systemctl enable chronyd
```


### 状態確認
```bash
systemctl status chronyd
```


## 1-5 同期確認

### 同期状態確認
```bash
chronyc sources -v
```


### 時刻確認
```bash
date
```

---


# 2 NTPクライアント設定

## 2-1 chronyインストール

### インストール
```bash
dnf install -y chrony
```


### 確認
```bash
rpm -qa | grep chrony
```


## 2-2 chrony設定ファイル編集

### バックアップ
```bash
cp -a /etc/chrony.conf /etc/chrony.conf.bak_$(date +%Y%m%d_%H%M%S)
```


### 確認
```bash
ls /etc/ | grep chrony
```


### 編集
```bash
vi /etc/chrony.conf
```


### コメントアウト
```bash
#sourcedir /run/chrony.d
```


### 追記
```bash
server <NTPサーバのIPアドレス> iburst
```


### 設定の意味

|設定|意味|
|---|---|
server|同期先NTPサーバ|
iburst|起動時高速同期|


## 2-3 chrony起動

### 起動
```bash
systemctl start chronyd
```


### 自動起動
```bash
systemctl enable chronyd
```


### 状態確認
```bash
systemctl status chronyd
```


## 2-4 同期確認
```bash
chronyc sources -v
```

### OKの目安
```bash
^* 172.31.xx.xx
```
*がついていること



## 2-5 タイムゾーン変更

### 現在確認
```bash
timedatectl
```


### タイムゾーン変更
```bash
timedatectl set-timezone Asia/Tokyo
```


### 確認
```bash
timedatectl
```

---

# 最終確認

以下を確認する


## NTPサーバ
```bash
chronyc sources -v
```


## NTPクライアント
```bash
chronyc sources -v
```

## 確認ポイント

|項目|確認内容|
|---|---|
NTPサーバ|AWS NTPと同期|
NTPクライアント|NTPサーバと同期|
同期状態|`*` 表示|
時刻|dateコマンドで一致|



# 最終構成
```bash
NTP Client
↓
NTP Server (chrony)
↓
AWS NTP
169.254.169.123
```

- サーバ間時刻同期
- AWS内部NTP利用
- クライアント時刻配布
- ログ時刻の統一
- 分散システムの整合性確保

---