# BigIP-1 演習フル手順（AMI作成〜分散確認）

---

## 概要

本演習では以下を実施する。

- Webサーバのベース機を作成
- AMIを作成し複製
- BigIPでNode / Pool / Virtual Serverを構築
- 分散方式を変更し動作確認

---

## 構成図
```bash
User
↓
EIP（BigIP Public）
↓
Virtual Server
↓
Pool（分散制御）
↓
Node（各Webサーバ）
```
---

## 0. Webサーバ作成（ベース機）

### 操作サーバ
EC2（Webベース機）

### この工程でしていること
- Webサーバの基本構成を作成
- 後でAMIとして複製するための元を作る

### Webインストール
```bash
sudo dnf install httpd -y
```
```bash
sudo systemctl enable httpd
```
```bash
sudo systemctl start httpd
```

### 識別用ページ作成
```bash
echo “表示したい内容” | sudo tee /var/www/html/index.html
```

### 確認方法
```bash
curl http://自分のPrivateIP
```

### OKの目安
- 作成したメッセージが表示される

---

## 1. AMI作成

### 操作場所
AWSコンソール

### この工程でしていること
- 同一構成のインスタンスを複製できるようにする

### 手順
- インスタンス → アクション → イメージとテンプレート → イメージを作成
- 名前例：web-base-ami-2026xxxx
- ステータスが available になるまで待つ

### OKの目安
- AMIが available

---

## 2. AMIからインスタンス作成

### 操作場所
AWSコンソール → 各自

### この工程でしていること
- 同一構成Webを複数起動
- BigIPの分散対象を用意

### 注意
- Privateサブネットに配置
- SGで80をBigIP Privateから許可

### 識別ページ変更
```bash
echo “表示したい内容” | sudo tee /var/www/html/index.html
```

### OKの目安
- curlで変更後表示

---

## 3. BigIP GUI接続

### 操作端末
ローカルPC

### この工程でしていること
- 踏み台経由でBigIP管理画面へアクセス
```bash
ssh -L 10443:172.31.2.20:443 -i 鍵.pem ec2-user@踏み台IP
```
ブラウザ：
```bash
https://localhost:10443
```
### OKの目安
- BigIPログイン画面表示

---

## 4. Node作成

### 操作場所
BigIP GUI

### この工程でしていること
- Webサーバ実体を登録

Local Traffic → Nodes → Create

| 設定 | 内容 |
|------|------|
| Name | web1 |
| Address | PrivateIP |

### CLI例
```bash
create ltm node web1 address 172.31.x.x
```
### OKの目安
- Status緑（Available）

---

## 5. Pool作成

### この工程でしていること
- 分散ロジックを設定

Local Traffic → Pools → Create

| 設定 | 内容 |
|------|------|
| Name | pool_web |
| Monitor | http |
| Method | Round Robin |

Members：
- web1:80
- web2:80

CLI例
```bash
create ltm pool pool_web members add { web1:80 web2:80 }
```
### OKの目安
- メンバー緑

---

## 6. Virtual Server作成

### この工程でしていること
- ユーザ受付窓口作成

Local Traffic → Virtual Servers → Create

| 設定 | 内容 |
|------|------|
| Name | vs_web |
| Destination | 172.31.2.20 |
| Port | 80 |
| Protocol | TCP |
| SNAT | AutoMap |
| Default Pool | pool_web |

CLI例
```bash
create ltm virtual vs_web destination 172.31.2.20:80 ip-protocol tcp pool pool_web source-address-translation { type automap }
```

### OKの目安
- VS緑

---

## 7. 分散確認

ブラウザ：
```bash
http://EIP
```
### OKの目安
- リロードで表示が交互になる（Round Robin）

---

## 8. 分散方式変更

Pool → Load Balancing Method変更

- Least Connections
- Ratio

### 確認
- アクセス偏りを確認

---

## 9. 障害確認

Web側：
```bash
sudo systemctl stop httpd
```
### OKの目安
- Poolメンバー赤
- 生存ノードのみへ振り分け

---

## 本質理解

| 用語 | 意味 |
|------|------|
| AMI | 複製元 |
| Node | 実体 |
| Pool | 分散制御 |
| Virtual Server | 受付 |
| AutoMap | 戻り通信整合 |

---

# BigIP-2 演習

---

## 構成図
```bash
User
↓
Virtual Server
↓
Pool（teamG1）
↓
Web × 複数
```
---

## 1. 分散確認

### 操作
- Pool作成（teamG1）
- Method：Round Robin
- VSにHTTP Profile追加

### OK
- 表示切替確認

---

## 2. iRule作成

### この工程でしていること
- 全停止時に503を返す制御
```bash
when HTTP_REQUEST {
if { [active_members teamG1] == 0 } {
HTTP::respond 503 content “

Sorry…


“ “Content-Type” “text/html”
}
}
```

### 重要
- HTTP ProfileがnoneだとHTTP_REQUEST使えない
- 必ずhttpに変更

---

## 3. VSへ適用

Local Traffic → Virtual Server → Resources → iRules追加

---

## 4. 確認

### 通常時
- 通常ページ表示

### 全停止時
- 503 Sorry表示

---

# BigIP-3 CLI演習

---

## 目的

- tmshでPool作成
- CLI操作理解

---

## 1. tmshへ移行
```bash
tmsh
```
OK：
- (tmos)# 表示

---

## 2. Pool作成
```bash
create ltm pool teamG2 
members add { 
CL_onishi_G1:80 
CL_yoshino_teamG1:80 
CL_koinuma_teamG1:80 
CL_ueda_teamG1:80 
CL_kosaka_teamG1:80 
} monitor http
```
### この工程でしていること
- 既存Nodeを参照
- HTTP監視付きPool作成

---

## 3. 作成確認
```bash
list ltm pool teamG2
```
```bash
show ltm pool teamG2 members
```
OK：
- members表示
- up状態

---

## 4. 設定保存
```bash
save sys config
```
### この工程でしていること
- 再起動後も設定保持

保存対象：
- /config/bigip.conf
- /config/bigip_base.conf
- /config/bigip_user.conf

---

## 完了状態

- 分散動作確認済み
- iRule制御理解
- CLI作成成功
- 設定保存完了