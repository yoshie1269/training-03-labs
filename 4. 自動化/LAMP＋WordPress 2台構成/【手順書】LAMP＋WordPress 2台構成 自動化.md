# LAMP＋WordPress 2台構成（Ansible）手順書

## 概要

### この構成でしていること

- WebサーバとDBサーバを分離
- Ansibleで自動構築
- WordPressを2tier構成で動作

---

### 構成図
```
Ansible実行サーバ  
↓ SSH  
Webサーバ → TCP/3306 → DBサーバ  
```

---

### サーバ役割

|役割|内容|
|---|---|
|Ansible|構築実行|
|Web|Apache / PHP / WordPress|
|DB|MariaDB|

---

## 1 事前準備

### この工程でしていること

構築用サーバと通信制御の準備

---

### EC2準備

- Ansibleサーバ  
- Webサーバ  
- DBサーバ  

---

### セキュリティグループ

#### Webサーバ

|Port|Source|
|---|---|
|22|自分IP|
|80|0.0.0.0/0|

---

#### DBサーバ

|Port|Source|
|---|---|
|22|自分IP|
|3306|WebサーバSG|

---

### OKの目安

- DBの3306はWebのみ許可  
- 0.0.0.0/0は禁止  

---

## 2 Ansibleインストール（実行サーバ）

### この工程でしていること

自動構築ツール導入

---

### コマンド
```bash
sudo dnf update -y  
```
```bash
sudo dnf install -y ansible-core 
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

## 3 SSH鍵作成

### この工程でしていること

パスワードなし接続準備

---

### コマンド
```bash
ssh-keygen -t ed25519  
```

---

### 保存先
```bash
~/.ssh/id_ed25519  
```

---

### 確認
```bash
ls -l ~/.ssh  
```

---

### OKの目安

- id_ed25519  
- id_ed25519.pub  

---

## 4 公開鍵配布（Web / DB）

### この工程でしていること

Ansibleから接続できるようにする

---

### 公開鍵確認
```bash
cat ~/.ssh/id_ed25519.pub  
```

---

### 接続先で実施
```bash
mkdir -p ~/.ssh  
```
```bash
chmod 700 ~/.ssh  
```

```bash
vi ~/.ssh/authorized_keys  
```
（公開鍵を貼り付け）  
```bash
chmod 600 ~/.ssh/authorized_keys  
```

---

### OKの目安

- パスワードなしSSH成功  

---

## 5 SSH接続確認

### コマンド
```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@WebPrivateIP  
```
```bash
ssh -i ~/.ssh/id_ed25519 ec2-user@DBPrivateIP  
```

---

### OKの目安

- 両方ログイン成功  

---

## 6 作業ディレクトリ作成

### この工程でしていること

Ansible管理用ディレクトリ作成

---

### コマンド
```bash
cd ~  
```
```bash
mkdir ansible  
```
```bash
cd ansible  
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
vi inventory.ini  
```

---

### 確認
```bash
cat inventory.ini  
```

---

### OKの目安

- IP・鍵設定が正しい  

---

## 8 疎通確認

### この工程でしていること

Ansible接続確認

---

### コマンド
```bash
ansible -i inventory.ini all -m ping  
```
---

### OKの目安

- SUCCESS表示  

---

## 9 playbook作成

### この工程でしていること

構築処理をコード化

---

### ファイル
```bash
vi lamp_wp_2tier.yml  
```
---

### 内容

- DB構築  
- Web構築  
- WordPress設定  

---

### OKの目安

- YAML構文が正しい  

---

## 10 依存モジュール導入

### この工程でしていること

MySQL操作用モジュール導入

---

### コマンド
```bash
ansible-galaxy collection install community.mysql  
```

---

### OKの目安

- インストール成功  

---

## 11 構文チェック

### この工程でしていること

playbookエラー事前検出

---

### コマンド
```bash
ansible-playbook -i inventory.ini lamp_wp_2tier.yml --syntax-check  
```

---

### OKの目安

- エラーなし  

---

## 12 playbook実行

### この工程でしていること

環境自動構築

---

### コマンド
```bash
ansible-playbook -i inventory.ini lamp_wp_2tier.yml  
```

---

### 入力

- DB名  
- DBユーザ  
- パスワード  
- DBサーバIP  

---

### OKの目安

- failed=0  

---

## 13 動作確認

### Apache確認
```bash
ansible -i inventory.ini web -m shell -a "systemctl status httpd"  
```

---

### MariaDB確認
```bash
ansible -i inventory.ini db -m shell -a "systemctl status mariadb"  
```

---

### DB接続確認
```bash
ansible -i inventory.ini web -m shell -a "mysql -u ユーザ -h DBIP -pパス DB名 -e 'SELECT 1;'"  
```

---

### OKの目安

- httpd active  
- mariadb active  
- SELECT 1 成功  

---

## 14 WordPress確認

### ブラウザ

http://WebサーバPublicIP/  

---

### OKの目安

- 初期セットアップ画面表示  

---

## 15 よくあるミス

### SSH接続不可

- 鍵未配布  
- 権限ミス  

---

### Ansibleエラー

- inventoryミス  
- module不足  

---

### DB接続エラー

- SG設定ミス  
- DB_HOSTミス  
- wp-config.php誤り  

---

## 完了状態

- Web / DB 分離構成  
- DB外部非公開  
- Ansibleで再現可能  
- WordPress正常起動  