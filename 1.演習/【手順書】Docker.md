# Amazon Linux 2023 + Docker + Docker Compose で WordPress 環境構築手順

---

# 0 構成概要

## この構成でしていること

- Web/AP と DB を分離したコンテナ構成
- WordPress公式イメージを使わず自作
- データをホスト側に保存して永続化

---

## 構成

### Web/APコンテナ
- ベース: Amazon Linux 2
- Apache + PHP + WordPress

### DBコンテナ
- MariaDB

---

## 永続化対象

|対象|保存場所|
|---|---|
DBデータ|/var/lib/mysql|
設定|wp-config.php|
コンテンツ|wp-content|

---
<br>

# 1 事前準備

## この工程でしていること

EC2へ接続し環境確認

---

## 接続
```bash
ssh -i ~/.ssh/CL_onishi.pem ec2-user@<EC2のパブリックIP>
```
---

## 確認
```bash
hostnamectl  
```
```bash
cat /etc/os-release  
```

---

## OKの目安

- OSが Amazon Linux 2023 と表示される  
- ホスト名が確認できる  

---
<br>

# 2 Dockerインストール

## この工程でしていること

コンテナ実行環境を準備

---

## インストール
```bash
sudo dnf update -y  
```
```bash
sudo dnf install -y docker  
```

---

## 起動
```bash
sudo systemctl enable --now docker  
```
---

## 権限付与
```bash
sudo usermod -aG docker ec2-user  
```
※再ログイン推奨

---

## 確認
```bash
docker --version  
```
```bash
systemctl status docker --no-pager  
```

---

## OKの目安

- docker version が表示される  
- status が active (running)  

---
<br>

# 3 Docker Composeインストール

## この工程でしていること

複数コンテナをまとめて管理

---

## インストール
```bash
mkdir -p ~/.docker/cli-plugins  
```
```bash
curl -SL https://github.com/docker/compose/releases/latest/download/docker-compose-linux-x86_64  
-o ~/.docker/cli-plugins/docker-compose  
```
```bash
chmod +x ~/.docker/cli-plugins/docker-compose  
```

---

## 確認
```bash
docker compose version  
```

---

## OKの目安

- docker compose version が表示される  

---
<br>

# 4 Buildx更新（エラー時のみ）

## この工程でしていること

docker build機能の拡張

---

## 更新
```bash
rm -f ~/.docker/cli-plugins/docker-buildx  
```

---

## 確認
```bash
docker buildx version  
```

---

## OKの目安

- バージョンが表示される  

---
<br>

# 5 作業ディレクトリ作成

## この工程でしていること

構成ファイル・データ保存場所を作成

---
```bash
mkdir -p ~/wordpress-lab/web  
```
```bash
mkdir -p ~/wordpress-lab/data/wordpress  
```
```bash
mkdir -p ~/wordpress-lab/data/db  
```

```bash
cd ~/wordpress-lab  
```

---

## OKの目安

- ディレクトリが作成されている  
```bash
ls -la ~/wordpress-lab  
```

---
<br>

# 6 Web/APコンテナ作成（Dockerfile）

## この工程でしていること

Amazon Linux 2ベースのWordPressコンテナを作成

---

## 確認

Dockerfileが存在すること  
```bash
ls ~/wordpress-lab/web  
```

---

## OKの目安

- Dockerfile が存在する  

---
<br>

# 7 永続化設定

## この工程でしていること

コンテナ削除後もデータを保持

---

## 作成
```bash
mkdir -p ~/wordpress-lab/data/wordpress/wp-content  
```
```bash
touch ~/wordpress-lab/data/wordpress/wp-config.php  
```

---

## 確認
```bash
ls -la ~/wordpress-lab/data/wordpress  
```

---

## OKの目安

- wp-config.php が存在  
- wp-content ディレクトリが存在  

---
<br>

# 8 docker-compose.yml作成

## この工程でしていること

WebとDBを連携させる

---

## 確認
```bash
ls ~/wordpress-lab  
```

---

## OKの目安

- docker-compose.yml が存在  

---
<br>

# 9 YAML構文確認

## この工程でしていること

設定ミスチェック

---
```bash
docker compose config  
```
---

## OKの目安

- エラーが出ない  
- services が表示される  

---

<br>

# 10 コンテナ起動

## この工程でしていること

環境を起動

---
```bash
docker compose up -d --build  
```
```bash
docker compose ps 
```

---

## OKの目安

- web / db が Up 状態  

例  
Up  

---

# 11 初回エラー対応（themes不足）

## 症状

テーマが存在しない

---

## 原因

wp-contentが空で上書きされた

---

## 対応

themesをコピー

---

## 確認
```bash
ls -la ~/wordpress-lab/data/wordpress/wp-content/themes  
```

---

## OKの目安

- themes ディレクトリが存在  
- 中にテーマファイルがある  

---

# 12 ブラウザ確認

## アクセス

http://<EC2のパブリックIP>

---

## OKの目安

- WordPress初期画面が表示される  

---

# 13 WordPress初期設定

## この工程でしていること

サイト初期設定

---

## OKの目安

- 管理画面にログインできる  

---

# 14 動作確認

## 確認項目

- 画面表示OK  
- ログインOK  
- 投稿作成OK  

---

## OKの目安

- 投稿が表示される  
- エラーが出ない  

---

# 15 永続化確認

## この工程でしていること

データがホスト側に保存されているか確認

---

## 確認

ls -la ~/wordpress-lab/data/wordpress  
ls -la ~/wordpress-lab/data/db  

---

## OKの目安

- wp-config.php 存在  
- wp-content 存在  
- DBファイル存在  

---

# 16 再作成テスト

## この工程でしていること

コンテナ削除後もデータ保持できるか確認

---

docker compose down  
docker compose up -d  

---

## OKの目安

- 初期設定画面に戻らない  
- 投稿が残っている  

---

# 17 永続化の考え方

## WordPress本体

イメージに含める  

---

## 設定

wp-config.php を永続化  

---

## コンテンツ

wp-content を永続化  

---

## DB

/var/lib/mysql を永続化  

---

# 最終構成

Client  
↓  
HTTP  
↓  
Apache（WordPress）  
↓  
MariaDB  

---

# まとめ

- WordPress公式イメージ未使用  
- Web/APとDBを分離  
- Amazon Linux 2ベースで構築  
- データ永続化済み  
- 再作成してもデータ保持可能  

---

# 発表用説明

本構成では、WordPress公式イメージを使用せず、  
Amazon Linux 2ベースのコンテナを自作しました。

Web/APとDBを分離し、  
DBデータ、設定ファイル、コンテンツをホストに保存することで、  
コンテナ再作成後もデータを保持できる構成としました。