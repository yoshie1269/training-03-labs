# Basic認証 構築手順書（Amazon Linux 2023）

## Basic認証とは

Webサイトや特定ディレクトリにユーザー名とパスワードによる制限をかける簡易な認証方式。

Amazon Linux 2023（AL2023）では、Apache（httpd）やNginxを用いて設定し、開発・テスト環境の一時的なセキュリティ対策やサイト公開前の保護に利用される。

- 仕組み：ブラウザが認証情報を要求し、サーバー側の.htpasswdファイルなどで照合
- 用途：テスト環境の保護、未公開サイトのアクセス制限

---
<br>

## 1 rootに昇格

### この工程でしていること
管理者権限で作業する

### コマンド
```bash
sudo su -
```

### 確認方法
```bash
whoami  
```

### OKの目安
- root と表示される  

---
<br>

## 2 Apache(httpd)をインストール

### この工程でしていること
WebサーバとBasic認証ツールをインストール

### コマンド
```bash
dnf install -y httpd httpd-tools  
```

### コマンドの意味
- httpd：Apache本体  
- httpd-tools：htpasswdなどの認証ツール  

### 確認方法
```bash
rpm -qa | grep httpd  
```

### OKの目安
- httpd が表示される  

---
<br>

## 3 Apacheを起動・自動起動設定

### この工程でしていること
Apacheを起動し常時動作させる

### コマンド
```bash
systemctl start httpd  
```
```bash
systemctl enable httpd 
```
```bash 
systemctl status httpd --no-pager  
```

### コマンドの意味
- start：起動  
- enable：自動起動設定  
- status：状態確認  

### OKの目安
- active (running)  

---
<br>

## 4 公開用ディレクトリと確認用ページ作成

### この工程でしていること
Basic認証対象のページを作成

### コマンド
```bash
mkdir -p /var/www/html/test
```

（HTMLファイル作成）

### 確認方法
```bash
ls -l /var/www/html/test
```

### OKの目安
- index.html が存在  

---
<br>

## 5 Basic認証設定ファイル作成

### この工程でしていること
特定ディレクトリに認証をかける

### 設定ファイル
```bash
/etc/httpd/conf.d/basic-test.conf  
```

### 設定内容
```bash
<Directory "/var/www/html/test">
    AuthType Basic
    AuthName "Basic Auth"
    AuthUserFile /etc/httpd/conf/.htpasswd
    Require valid-user
</Directory>
```

---

### 設定の意味

- AuthType Basic：Basic認証を使用  
- AuthName：認証画面の表示名  
- AuthUserFile：ユーザー情報ファイル  
- Require valid-user：登録ユーザーのみ許可  

---

### 確認方法
```bash
cat /etc/httpd/conf.d/basic-test.conf  
```

### OKの目安
- 設定内容が表示される  

---
<br>

## 6 Basic認証ユーザー作成

### この工程でしていること
認証用ユーザーを登録

### コマンド
```bash
htpasswd -c /etc/httpd/conf/.htpasswd <ユーザ名>  
```

### コマンドの意味
- -c：新規作成（初回のみ）  
- ユーザー名  

### 確認方法
```bash
cat /etc/httpd/conf/.htpasswd  
```

### OKの目安
- test:xxxx（暗号化パスワード）が表示  

---
<br>

## 7 設定ファイル確認

### この工程でしていること
設定内容の最終確認

### コマンド
```bash
cat /etc/httpd/conf.d/basic-test.conf  
```
```bash
cat /etc/httpd/conf/.htpasswd  
```
```bash
ls -l /var/www/html/test/index.html  
```

### OKの目安
- すべてのファイルが存在  

---
<br>

## 8 構文チェック

### この工程でしていること
設定ミスを確認

### コマンド
```bash
apachectl configtest  
```

### OKの目安
- Syntax OK  

---
<br>

## 9 Apache再起動

### この工程でしていること
設定を反映

### コマンド
```bash
systemctl restart httpd  
```
```bash
systemctl status httpd --no-pager  
```

### OKの目安
- active (running)  

---
<br>

## 10 動作確認（重要）

### この工程でしていること
認証が正しく動作するか確認

---

### 未認証アクセス
```bash
curl -I http://127.0.0.1/test/  
```

### OKの目安
- HTTP/1.1 401 Unauthorized  

---

### 認証ありアクセス
```bash
curl -u test:<パスワード> -I http://127.0.0.1/test/  
```

### OKの目安
- HTTP/1.1 200 OK  

---
<br>

## 11 ブラウザ確認

### この工程でしていること
実際の画面確認

---

### URL

http://サーバIP/test/  

---

### OKの目安

- 認証画面が表示  
- ログイン後に Hello World 表示  

---
<br>

## 12 パスワード再設定

### この工程でしていること
既存ユーザーのパスワード変更

### コマンド
```bash
htpasswd /etc/httpd/conf/.htpasswd test  
```

### OKの目安
- 新しいパスワードでログインできる  

---
<br>

## 13 ユーザー名確認

### この工程でしていること
登録ユーザー確認

### コマンド
```bash
cat /etc/httpd/conf/.htpasswd  
```

### OKの目安
- ユーザー名が表示される  

---
<br>

## 14 よくあるハマりポイント

### アクセスURLミス
- /test/ にアクセスする必要あり  

---

### ファイル未作成
- index.html が存在しない  

---

### 認証が効かない
- 200 OK のまま → 設定ミス  

---

### 正常な挙動

- 未認証 → 401  
- 認証あり → 200  

---

### htpasswdの注意

- -c を付けると上書きされる  
- 2回目以降は付けない  