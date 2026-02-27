# SRX演習 SRX-1（Web公開：事前準備〜DNAT〜外部疎通）完全手順書

---

## 概要

- 事前準備（AMI作成、AMIからWebサーバ起動）
- SRX に EIP を割り当てる（セカンダリIP＋EIP関連付け）
- SRX で DNAT（宛先NAT）設定
- 外部から `curl http://EIP` で到達確認

---

## 構成図
```bash
[外部端末]
|
|  HTTP(80)  宛先=EIP
v
[EIP] -> [SRX untrust 側(セカンダリIP:172.31.0.99)]
|
| DNAT：宛先を Web へ変換
v
[Web(172.31.1.251:80) trust側]
```

---

## 前提（すでに出来ている想定）

- VPC / サブネット / ルーティングは講師構成どおり
- 踏み台（STEP）サーバ作成済み
- SRX作成済み
- SRX インターフェース設定済み
- SRX セキュリティポリシー設定済み

### SRX インターフェース（復習・参照）
```bash
set interfaces ge-0/0/0 unit 0 family net address 172.31.0.10/24
```
```bash
set interfaces ge-0/0/1 unit 0 family net address 172.31.1.10/24
```
```bash
commit check
```
```bash
commit
```

---

# 0. 事前準備（値を決めてメモする）

## この工程でしていること
- 後工程で入力するIPや名前を固定し、作業中のミスを減らす

## この演習で使う値（例）

| 項目 | 値（例） | 説明 |
|------|----------|------|
| Web(Private IP) | 172.31.1.251 | trust側サブネットのWeb |
| SRX untrust IF | ge-0/0/0.0 | 外側IF |
| SRX trust IF | ge-0/0/1.0 | 内側IF |
| SRX untrust 既存IP | 172.31.0.10 | SRX外側 |
| SRX untrust セカンダリIP | 172.31.0.99 | EIP紐づけ先（入口） |
| EIP | （払い出し値） | 外部からアクセスするIP |
| DNAT pool名 | pool_F | Web転送先 |
| DNAT rule名 | TeamF_A | EIP側宛先一致ルール |

---

# 1. 事前準備（AMI作成用のベースWebサーバを作る）

## 操作場所
- AWSコンソール（EC2）
- Webベース機（OS操作）

## この工程でしていること
- Apache入りの「ベース機」を作って AMI 化する

---

## 1-1. EC2（ベース機）を起動する

### AWSコンソールで実施
- EC2 → インスタンス → インスタンス起動
- AMI：Amazon Linux 2023
- サブネット：講師指定（どちらでも良いが後でAMI化）
- SG：以下を許可
  - 22：自分のIP（SSH用）
  - 80：同一VPC内から（後でBigIP/SRXから来る想定）

### OKの目安
- インスタンスが running
- Public/Private IP が確認できる

---

## 1-2. ベース機へSSH接続する

### ローカル端末で実施（例）
```bash
ssh -i 鍵.pem ec2-user@ベース機のPublicIP
```
### OKの目安
- ログイン成功し `ec2-user@...$` になる

---

## 1-3. Apacheをインストールして起動する

## この工程でしていること
- AMIに入れるWebサーバソフトを準備する
```bash
sudo dnf install httpd -y
```
```bash
sudo systemctl enable httpd
```
```bash
sudo systemctl start httpd
```

### 確認方法
```bash
sudo systemctl status httpd
```
OKの目安：
- active (running)

---

# 2. AMI を作成する

## 操作場所
AWSコンソール（EC2）

## この工程でしていること
- Apache入りWebサーバを複製できるようにする

### 手順
- EC2 → インスタンス（ベース機選択）
- アクション → イメージとテンプレート → イメージを作成
- 名前をつけて作成

### 確認方法
- EC2 → AMI → 該当AMI
- Status：available になるまで待つ

OK：
- available

---

# 3. AMI から Webサーバ（本番用）を起動する（Private Subnet）

## 操作場所
AWSコンソール（EC2）

## この工程でしていること
- SRXが転送する先となるWebサーバをPrivateに作成する

---

## 3-1. AMIからインスタンスを起動

### 手順
- EC2 → AMI → 作成したAMI選択 → インスタンス起動
- サブネット：trust側（172.31.1.0/24 側）Private Subnet
- Private IP：起動後に確認してメモ
- SG：HTTP(80)を許可
  - 推奨：Source を SRX trust側（172.31.1.10/32）または trustサブネット（172.31.1.0/24）に限定
  - SSH(22)：踏み台（STEP）のSGまたは踏み台IPから許可

### OKの目安
- running

---

## 3-2. 踏み台からWebへSSH接続し Apache確認

### 踏み台に接続
```bash
ssh -i 鍵.pem ec2-user@踏み台PublicIP
```

### 踏み台 → Webへ接続
```bash
ssh -i 鍵.pem ec2-user@172.31.1.251
```

### Apache確認（AMIから起動しているので基本入っている想定）
```bash
sudo systemctl status httpd
```
```bash
sudo systemctl start httpd
```
```bash
sudo systemctl enable httpd
```

OK：
- active (running)
- enabled

---

# 4. SRX に EIP を割り当てる（セカンダリIP＋EIP関連付け）

## 操作場所
AWSコンソール（EC2）

## この工程でしていること
- 外部から入ってくる入口（EIP）をSRXに持たせる
- その入口の宛先IPをDNAT条件に使う

---

## 4-1. SRX の Elastic Network Interface（ENI）を特定する

### 手順
- EC2 → インスタンス → SRXインスタンス選択
- ネットワーク → ネットワークインターフェイス（ENI）を確認
- untrust 側ENI（172.31.0.10 が付いている方）を選ぶ

OK：
- 172.31.0.10 が Primary として見える

---

## 4-2. untrust ENI にセカンダリPrivate IPを追加する

### 手順
- 対象ENI → アクション → IPアドレスを管理
- セカンダリIPv4アドレスを追加
- 172.31.0.99 を追加（指定どおり）

OK：
- Secondary Private IP に 172.31.0.99 が表示

---

## 4-3. 172.31.0.99 に EIP を関連付ける

### 手順
- EC2 → Elastic IP → Elastic IP を割り当て
- アクション → Elastic IP を関連付ける
  - リソース：ネットワークインターフェイス（ENI）
  - プライベートIP：172.31.0.99
- 関連付け

OK：
- EIP が SRX ENI の 172.31.0.99 に紐づいている

---

# 5. SRX で DNAT（宛先NAT）を設定する

## 操作場所
- 踏み台（STEP）
- SRX（Junos）

## この工程でしていること
- EIP宛の通信（実体は172.31.0.99宛）を検知して
- 宛先を Web（172.31.1.251）へ書き換える

---

## 5-1. 踏み台から SRX へ接続
```bash
ssh -i 鍵.pem ec2-user@172.31.0.10
```
OK：
- Junosのプロンプト（例：`ec2-user@NW-FW>`）

---

## 5-2. 設定モードへ
```bash
configure
```
OK：
- `#` が付く（例：`ec2-user@NW-FW#`）

---

## 5-3. DNAT用 pool 作成（転送先＝Web）
```bash
set security nat destination pool pool_F address 172.31.1.251/32
```

### 何をしているか
- pool_F に「DNAT後の宛先（Web）」を登録する
- /32 は単一IP指定

### 確認（任意）
```bash
show security nat destination pool
```
OK：
- pool_F が表示される

---

## 5-4. DNATルール定義（EIP側宛先IPに一致）
```bash
set security nat destination rule-set 1 rule TeamF_A match destination-address 172.31.0.99/32
```

### 何をしているか
- 「宛先が 172.31.0.99 の通信」をDNAT対象として捕まえる
- 172.31.0.99 は EIP の紐づけ先（SRX側の入口）

---

## 5-5. DNAT先 pool を指定
```bash
set security nat destination rule-set 1 rule TeamF_A then destination-nat pool pool_F
```

### 何をしているか
- 捕まえた通信の宛先を pool_F（=172.31.1.251）へ変換する

---

## 5-6. commit
```bash
commit
```

OK：
- commit complete

---

## 5-7. DNAT設定確認
```bash
show security nat destination
```
OK：
- pool_F が存在
- rule-set 1 に TeamF_A が存在
- destination-address 172.31.0.99 と pool_F が紐づいている

---

# 6. 外部から動作確認

## 操作場所
外部端末（自分のPC等）

## この工程でしていること
- インターネット → EIP → SRX → DNAT → Web の到達確認
```bash
curl http://EIP

```
OK：
- Webのindex（This is Web NODE ...）が返る

---

# 7. トラブル時チェック（見たまま確認）

## 7-1. Webが動いているか（Webサーバ側）
```bash
sudo systemctl status httpd
```
```bash
curl http://localhost
```
OK：
- active (running)
- indexが返る

---

## 7-2. SRXにDNATが入っているか（SRX側）
```bash
show security nat destination
```
OK：
- pool_F と TeamF_A が存在

---

## 7-3. EIPの紐づけが正しいか（AWS側）

- EIP → SRX untrust ENI
- Private IP = 172.31.0.99

OK：
- 間違いなく 172.31.0.99

---

## 7-4. ポリシーで落ちていないか（SRX側）
```bash
show security policies
```
OK：
- untrust→trust の許可が存在

---

# 完了状態

- Web（172.31.1.251）が稼働し index が返る
- SRX untrust に 172.31.0.99 を追加済
- 172.31.0.99 に EIP が関連付け済
- SRX に DNAT（172.31.0.99 → 172.31.1.251）設定済
- 外部から `curl http://EIP` が成功する