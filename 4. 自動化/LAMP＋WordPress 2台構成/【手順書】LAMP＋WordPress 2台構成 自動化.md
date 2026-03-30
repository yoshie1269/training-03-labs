# LAMP＋WordPress 2台構成 自動化手順書（Ansible・簡略版）

## 概要

### この構成でしていること

- WebサーバとDBサーバを分離
- Ansibleで構築を自動化
- WordPress環境を再現可能にする

---

### 構成

[Ansible実行端末]  
↓ SSH  
[EC2-Web]　　　[EC2-DB]  

- Web：Apache / PHP / WordPress  
- DB：MariaDB  

---

## 事前準備

### この工程でしていること

Ansibleから各サーバへ接続可能な状態を作る

---

### 実施内容

- Ansible実行端末を用意  
- Web / DBへSSH接続設定  
- 鍵ファイル準備  
- セキュリティグループ設定  

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
|3306|SG-Web|

---

### OKの目安

- DBの3306がWebのみ許可  
- 0.0.0.0/0で3306開いていない  

---

## 作成ファイル

### この工程でしていること

Ansibleで使用する設定ファイルを用意

---

### 作成内容

- inventory.ini ファイルを作成  
- lamp_wp_2tier.yml ファイルを作成  

---

## Ansible実行端末準備

### この工程でしていること

Ansibleの実行環境を整備

---

### 確認

ansible --version  

---

### OKの目安

- バージョンが表示される  

---

### コレクション導入

ansible-galaxy collection install community.mysql  

---

### OKの目安

- community.mysql が導入される  

---

### Pythonモジュール

pip install PyMySQL  

---

### OKの目安

- エラーなし  

---

## inventory.ini準備

### この工程でしていること

管理対象サーバを定義

---

### 実施内容

- Webサーバ情報を記載  
- DBサーバ情報を記載  
- SSH鍵 / become設定記載  

---

### 確認

ansible -i inventory.ini all -m ping  

---

### OKの目安

- web / db 両方 SUCCESS  

---

## playbook準備

### この工程でしていること

構築処理を定義

---

### 実施内容

- lamp_wp_2tier.yml ファイルを作成  
- DBサーバ用Playを定義  
- Webサーバ用Playを定義  
- vars_promptを設定  

---

### 入力項目

- DB名  
- DBユーザ名  
- DBパスワード  
- DBサーバPrivateIP  

---

## DBサーバ側処理

### この工程でしていること

MariaDBとWordPress用DB構築

---

### 自動化内容

- MariaDBインストール  
- 起動・自動起動設定  
- 設定ファイル作成  
- 再起動  
- DB作成  
- ユーザ作成  
- 権限付与  

---

### 確認項目

- MariaDB active  
- DB存在  
- ユーザ存在  
- 権限付与済  

---

### OKの目安

- systemctl status mariadb → active  
- mysqlログイン可能  

---

## Webサーバ側処理

### この工程でしていること

Apache + WordPress構築

---

### 自動化内容

- パッケージ更新  
- Apache / PHP インストール  
- Apache起動  
- WordPressダウンロード  
- 展開・配置  
- wp-config.php 作成  
- DB接続情報設定  
- Apache設定変更  
- 権限設定  
- 構文チェック  
- DB疎通確認  
- Apache再起動  

---

### 確認項目

- httpd active  
- wp-config.php構文OK  
- httpd -t OK  
- DB接続成功  

---

### OKの目安

- systemctl status httpd → active  
- php -l wp-config.php → OK  

---

## 実行方法

### この工程でしていること

Ansibleで自動構築実行

---

### コマンド

ansible-playbook -i inventory.ini lamp_wp_2tier.yml  

---

### 入力内容

- DB名  
- DBユーザ  
- DBパスワード  
- DBのPrivateIP  

---

## 実行後確認

### Apache状態確認

ansible -i inventory.ini web -m shell -a "systemctl status httpd"  

---

### OKの目安

- active (running)  

---

### MariaDB状態確認

ansible -i inventory.ini db -m shell -a "systemctl status mariadb"  

---

### OKの目安

- active (running)  

---

### Web → DB接続確認

ansible -i inventory.ini web -m shell -a "mysql -u <ユーザ> -h <DB_IP> -p'<パスワード>' <DB名> -e 'SELECT 1;'"  

---

### OKの目安

- SELECT 1 が返る  

---

### ブラウザ確認

http://WebサーバPublicIP/  

---

### OKの目安

- WordPressセットアップ画面  

---

## 典型ミス

### DB接続エラー

原因  
- SG設定ミス  
- DB IPミス  
- 認証情報ミス  

---

### Apache表示されない

原因  
- 80未開放  
- httpd未起動  

---

### WordPress表示されない

原因  
- 配置ミス  
- 権限ミス  
- wp-config誤り  

---

### Ansibleエラー

原因  
- community.mysql未導入  
- PyMySQL未導入  
- inventory設定ミス  

---

## 完成状態

- Web/DB分離構成  
- DB非公開（内部のみ）  
- Apache + PHP + WordPress動作  
- Web → DB疎通OK  
- WordPress初期画面表示  