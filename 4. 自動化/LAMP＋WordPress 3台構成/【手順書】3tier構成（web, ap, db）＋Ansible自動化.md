# 3tier構成（Web / AP / DB）＋Ansible自動化 手順書

## 概要

### この構成でしていること

- Web / AP / DB を分離した3tier構成
- Ansibleで自動構築
- セキュアなWordPress環境を構築

---

### 構成図
```bash
Internet  
↓ 80  
Webサーバ（Apache）  
↓  
APサーバ（WordPress）  
↓  
DBサーバ（MariaDB）  
```

---

## 1 EC2構築

### この工程でしていること

3tier構成の基盤を用意

---

### 作成サーバ

- Webサーバ  
- APサーバ  
- DBサーバ  

---

### 配置

- 同一VPC  
- 同一サブネット  

---

### OKの目安

- 3台すべて起動  
- PrivateIPで相互通信可能  

---

## 2 セキュリティグループ設定

### この工程でしていること

最小権限で通信制御

---

### Webサーバ

|Port|Source|
|---|---|
|22|自分IP|
|80|0.0.0.0/0|

---

### APサーバ

|Port|Source|
|---|---|
|22|自分IP|
|80|WebサーバSG|

---

### DBサーバ

|Port|Source|
|---|---|
|22|自分IP|
|3306|APサーバSG|

---

### OKの目安

- DBは外部公開されていない  
- APもWeb以外からアクセス不可  

---

## 3 Ansibleサーバ準備

### この工程でしていること

自動構築環境準備

---

### コマンド
```bash
dnf install ansible -y  
```

---

### 確認
```bash
ansible --version  
```

---

### OKの目安

- バージョン表示  

---

## 4 SSH鍵作成

### この工程でしていること

パスワードレス接続準備

---

### コマンド
```bash
ssh-keygen  
```
---

### OKの目安

- ~/.ssh/id_ed25519 作成  

---

## 5 公開鍵配布

### この工程でしていること

Ansibleから各サーバへ接続可能にする

---

### コマンド
```bash
ssh-copy-id -i ~/.ssh/id_ed25519.pub ec2-user@AP  
ssh-copy-id -i ~/.ssh/id_ed25519.pub ec2-user@DB  
ssh-copy-id -i ~/.ssh/id_ed25519.pub ec2-user@Web  
```

---

### OKの目安

- パスワードなしSSH成功  

---

## 6 作業ディレクトリ作成

### この工程でしていること

Ansible管理用ディレクトリ準備

---

### コマンド
```bash
mkdir ~/ansible  
```
```bash
cd ~/ansible  
```

---

### OKの目安

- ansibleディレクトリ存在  

---

## 7 inventoryファイル作成

### この工程でしていること

対象サーバ定義

---

### ファイル
```bash
inventory_3tier.ini  
```

---

### 確認
```bash
cat inventory_3tier.ini  
```

---

### OKの目安

- 3台すべて定義されている  

---

## 8 Playbook作成

### この工程でしていること

構築処理をコード化

---

### ファイル
```bash
lamp_wp_3tier.yml  
```

---

### 内容

- DB構築  
- AP構築  
- Web構築  
- WordPress設定  

---

### OKの目安

- YAML構文が正しい  

---

## 9 疎通確認

### この工程でしていること

Ansible接続確認

---

### コマンド
```bash
ansible -i inventory_3tier.ini all -m ping  
```

---

### OKの目安

- 全サーバ SUCCESS  

---

## 10 Playbook実行

### この工程でしていること

3tier環境を自動構築

---

### コマンド
```bash
ansible-playbook -i inventory_3tier.ini lamp_wp_3tier.yml  
```

---

### 入力内容

- DB名  
- DBユーザ  
- DBパスワード  
- Web公開URL  

---

### OKの目安

- failed=0  

---

## 11 構築内容（自動化）

### DBサーバ

#### この工程でしていること

データベース環境構築

---

#### 内容

- MariaDBインストール  
- 起動 / 自動起動  
- 外部接続設定  
- DB作成  
- ユーザ作成 / 権限付与  

---

### APサーバ

#### この工程でしていること

アプリケーション構築

---

#### 内容

- Apache + PHPインストール  
- WordPress配置  
- wp-config.php作成  
- DB接続設定  
- URL設定  

---

### Webサーバ

#### この工程でしていること

フロントサーバ構築

---

#### 内容

- Apacheインストール  
- リバースプロキシ設定  
- Web → AP転送  

---

## 12 動作確認

### この工程でしていること

サービス全体確認

---

### ブラウザ
```bash
http://WebサーバPublicIP  
```

---

### OKの目安

- WordPress初期画面表示  
- CSS崩れなし  

---

## 13 疎通確認（任意）

### Web → AP
```bash
curl http://APのPrivateIP  
```

---

### OKの目安

- HTML応答あり  

---

### AP → DB
```bash
mysql -u ユーザ -h DBPrivateIP -p DB名  
```

---

### OKの目安

- 接続成功  

---

## 14 完成状態

### この構成で実現できていること

- Web / AP / DB 分離  
- セキュリティ最適化  
- WordPress正常動作  
- Ansibleで再現可能  

---

## 重要ポイント

### 3tier構成の本質

- Web：受付  
- AP：処理  
- DB：データ  

---

### メリット

- スケーラビリティ  
- セキュリティ向上  
- 障害切り分け容易  

---

## 完了

- WordPressがブラウザで表示される  
- Ansibleで再構築可能  