# APサーバ構築 手順書

# ■ ap1：JDK 導入・環境変数設定

## 概要

本手順書では、AP サーバ（Tomcat 等）を稼働させるために必要な  
Java 実行環境（JDK：Amazon Corretto）を導入・設定する。

本工程により、以下を実現する。

- Java 実行環境の導入
- JDK の標準配置
- 環境変数 PATH の設定
- Java コマンドの有効化

## 前提条件

- Amazon Linux 環境であること
- インターネット接続が可能であること
- root 権限が使用できること
- /opt ディレクトリが利用可能であること


## 1-1. root ユーザーへ切り替え、作業ディレクトリへ移動する

### この工程でしていること
- 管理者権限で作業できるようにする
- ダウンロード用の作業場所へ移動する


### 実行コマンド
```bash
sudo su -
```
```bash
cd /home/ec2-user
```


### コマンドの意味
- sudo su -
  
  root 権限へ切り替える
- cd
  
   作業ディレクトリ移動


### 確認方法とOKの目安
- プロンプトが # になる
- pwd で /home/ec2-user が表示される


## 1-2. Amazon Corretto（JDK）をダウンロードする


### この工程でしていること
- Java 開発・実行環境を公式サイトから取得する


### 実行コマンド
```bash
wget https://corretto.aws/downloads/latest/amazon-corretto-8-x64-linux-jdk.tar.gz
```


### コマンドの意味
- wget
  
  ファイルをインターネットから取得する


### 確認方法とOKの目安
```bash
ls -l amazon-corretto-8-x64-linux-jdk.tar.gz
```
- ファイルサイズが 0 でない
- ファイルが存在する


## 1-3. JDK 圧縮ファイルを展開する


### この工程でしていること
- ダウンロードした JDK を利用可能な形にする


### 実行コマンド
```bash
tar zxf amazon-corretto-8-x64-linux-jdk.tar.gz
```

### コマンドの意味

| オプション | 内容 |
|------------|------|
| z | gzip 解凍 |
| x | 展開 |
| f | ファイル指定 |


### 確認方法とOKの目安
```bash
ls
```
- amazon-corretto-8.482.08.1-linux-x64 などのディレクトリが生成されている


## 1-4. JDK ディレクトリを /opt に移動する


### この工程でしていること
- アプリケーション用標準配置場所に移動する
- 管理しやすくする


### 実行コマンド
```bash
mv amazon-corretto-8.482.08.1-linux-x64 /opt
```


### コマンドの意味
- mv
  
  ファイル・ディレクトリ移動


### 確認方法とOKの目安
```bash
ls /opt | grep corretto
```
- JDK ディレクトリが表示される


## 1-5. bash 設定ファイルをバックアップする


### この工程でしていること
- 環境変数設定前に元ファイルを保存する
- 設定失敗時に復旧できるようにする


### 実行コマンド
```bash
cp /root/.bash_profile /root/.bash_profile.bak
```

### コマンドの意味
- cp
  
  ファイルコピー
- .bak
  
  バックアップ識別


### 確認方法とOKの目安
```bash
ls -a /root | grep bash_profile
```
- .bak ファイルが存在する


## 1-6. PATH に JDK を追加する


### この工程でしていること
- java / javac コマンドをどこからでも実行可能にする
- 環境変数を設定する


### 実行コマンド
```bash
vi /root/.bash_profile
```


### 追記内容
```bash
PATH=$PATH:$HOME/bin:/opt/amazon-corretto-8.482.08.1-linux-x64/bin
```

### 設定の意味
- /opt/.../bin を検索対象に追加
- java コマンドを有効化

### 確認方法とOKの目安
- 重複記載がない
- パスが正しい


## 1-7. 設定を反映する


### この工程でしていること
- 編集した環境変数を反映させる
- 再ログインで設定を有効化する


### 実行コマンド
```bash
exit
```
```bash
sudo su -
```

### コマンドの意味
- exit
  
  ログアウト
- sudo su -
  
  再ログイン


### 確認方法とOKの目安
```bash
java -version
```
- Amazon Corretto のバージョンが表示される
- エラーが出ない


## 1-8. ここまでで完了していること

- JDK が正常に配置されている
- PATH が設定されている
- java コマンドが使用可能
- AP サーバ導入準備が完了している

<br>

---

# ■ ap2：Tomcat インストール・サービス化

## 概要

本手順書では、Java 環境上に Apache Tomcat を導入し、  
AP サーバとして常時稼働できる状態に設定する。

本工程により、以下を実現する。

- Tomcat 専用ユーザー作成
- Tomcat 本体導入
- 環境変数設定
- 自動デプロイ無効化
- systemd 登録
- 自動起動設定


## 前提条件

- JDK（Amazon Corretto）が導入済みであること
- /opt に JDK が存在すること
- root 権限で作業していること
- インターネット接続可能であること


## 2-1. Tomcat 専用ユーザーを作成する


### この工程でしていること
- Tomcat 専用の実行ユーザーを作成する
- OS へのログインを禁止し、セキュリティを確保する


### 実行コマンド
```bash
useradd -s /sbin/nologin tomcat
```

### コマンドの意味
- useradd：ユーザー作成
- -s /sbin/nologin：ログイン禁止


### 確認方法とOKの目安
```bash
id tomcat
```
- tomcat ユーザーが存在する


## 2-2. Tomcat 本体をダウンロードする

### この工程でしていること
- Apache 公式サイトから Tomcat を取得する


### 実行コマンド
```bash
wget https://dlcdn.apache.org/tomcat/tomcat-9/v9.0.115/bin/apache-tomcat-9.0.115.tar.gz
```


### コマンドの意味
- wget：ファイル取得


### 確認方法とOKの目安
```bash
ls -l apache-tomcat-9.0.115.tar.gz
```
- ファイルが存在する


## 2-3. Tomcat を展開する


### この工程でしていること
- Tomcat を利用可能な状態にする


### 実行コマンド
```bash
tar zxf apache-tomcat-9.0.115.tar.gz
```


### コマンドの意味
- zxf：gzip 解凍


### 確認方法とOKの目安
```bash
ls
```
- apache-tomcat-9.0.115 ディレクトリが存在


## 2-4. Tomcat を /usr/local に配置する


### この工程でしていること
- 標準配置場所へ移動し管理しやすくする


### 実行コマンド
```bash
cd /usr/local
```
```bash
mv /home/ec2-user/apache-tomcat-9.0.115 ./
```


### コマンドの意味
- mv：移動


### 確認方法とOKの目安
```bash
ls /usr/local | grep tomcat
```
- ディレクトリが存在


## 2-5. 所有者を tomcat に変更する

### この工程でしていること
- Tomcat を専用ユーザーで管理させる


### 実行コマンド
```bash
chown -R tomcat:tomcat apache-tomcat-9.0.115
```

### コマンドの意味
- chown -R：再帰的に所有者変更


### 確認方法とOKの目安
```bash
ls -ld /usr/local/apache-tomcat-9.0.115
```
- tomcat tomcat になっている


## 2-6. シンボリックリンクを作成する


### この工程でしていること
- バージョン変更時の切替を容易にする


### 実行コマンド
```bash
ln -s apache-tomcat-9.0.115 tomcat
```

### コマンドの意味
- ln -s：リンク作成


### 確認方法とOKの目安
```bash
ls -l /usr/local | grep tomcat
```
- tomcat → apache-tomcat-9.x.x

## 2-7. setenv.sh を作成する


### この工程でしていること
- Tomcat 起動時の Java 設定を定義する

### 実行コマンド
```bash
vi /usr/local/tomcat/bin/setenv.sh
```

### 設定内容
```bash
#!/bin/sh
CATALINA_HOME=/usr/local/tomcat
JAVA_HOME=/opt/amazon-corretto-8.482.08.1-linux-x64
JAVA_OPTS=”-Xms128m -Xmx521m”
```

### 設定の意味

| 項目 | 内容 |
|------|------|
| CATALINA_HOME | Tomcat ルート |
| JAVA_HOME | JDK |
| JAVA_OPTS | JVM メモリ |


### 確認方法とOKの目安
```bash
ls -l /usr/local/tomcat/bin/setenv.sh
```
- ファイルが存在


## 2-8. server.xml をバックアップする


### この工程でしていること
- 設定変更前に退避する


### 実行コマンド
```bash
cp /usr/local/tomcat/conf/server.xml /usr/local/tomcat/conf/server.xml.bak
```


### 確認方法
```bash
ls /usr/local/tomcat/conf | grep server
```
- .bak が存在


## 2-9. 自動デプロイ設定を変更する


### この工程でしていること
- 勝手なアプリ配置を防止する
- 運用を安定化させる


### 実行コマンド
```bash
vi /usr/local/tomcat/conf/server.xml
```

### 変更内容
```bash
unpackWARs=“false” autoDeploy=“false”
```

### 意味
- 自動展開無効
- 勝手に反映されない


### 確認方法
- false になっている


## 2-10. Tomcat サービスを登録する


### この工程でしていること
- systemd に Tomcat を登録する
- OS 起動と連動させる


### 実行コマンド
```bash
vi /etc/systemd/system/tomcat.service
```

### 設定内容
```bash
[Unit]
Description=Apache Tomcat 9
After=network.target

[Service]
User=tomcat
Group=tomcat
Type=oneshot
PIDFile=/usr/local/tomcat/tomcat.pid
RemainAfterExit=yes

ExecStart=/usr/local/tomcat/bin/startup.sh
ExecStop=/usr/local/tomcat/bin/shutdown.sh
ExecReStart=/usr/local/tomcat/bin/shutdown.sh;/usr/local/tomcat/bin/startup.sh

[Install]
WantedBy=multi-user.target
```

### 設定の意味
- User/Group：実行ユーザー
- ExecStart：起動処理
- ExecStop：停止処理


## 2-11. サービスファイルの権限設定


### この工程でしていること
- systemd が読み込めるようにする


### 実行コマンド
```bash
chmod 755 /etc/systemd/system/tomcat.service
```

### 確認方法
```bash
ls -l /etc/systemd/system/tomcat.service
```
- -rwxr-xr-x


## 2-12. Tomcat を自動起動登録・起動する


### この工程でしていること
- Tomcat を常駐化する


### 実行コマンド
```bash
systemctl enable tomcat
```
```bash
systemctl start tomcat
```

### コマンドの意味
- enable：自動起動
- start：起動


### 確認方法とOKの目安
```bash
systemctl status tomcat
```
- active (running)


## 2-13. ここまでで完了していること

- Tomcat が導入されている
- 専用ユーザーで稼働している
- Java と連携済み
- systemd 登録完了
- 自動起動設定完了

<br>

---

# ■ ap3：Apache 連携・アプリ配備・動作確認

## 概要

本手順書では、Apache と Tomcat を連携させ、  
Java Web アプリケーション（knowledge）を公開する設定を行う。

本工程により、以下を実現する。

- Apache から Tomcat へのリバースプロキシ設定
- Web アプリの配備
- Tomcat 実行権限設定
- ブラウザからのアクセス確認



## 前提条件

- JDK が導入済みであること
- Tomcat が稼働していること
- systemd 登録が完了していること
- インターネット通信が可能であること
- ポート 80 / 8080 が開放されていること


## 3-1. Apache（httpd）をインストールする


### この工程でしていること
- フロント Web サーバとして Apache を導入する
- Tomcat との中継役を準備する


### 実行コマンド
```bash
dnf install httpd
```


### コマンドの意味
- dnf install
  - パッケージ導入
- httpd
  - Apache 本体


### 確認方法とOKの目安
```bash
dnf list installed | grep httpd
```
- httpd が表示される


## 3-2. Apache にリバースプロキシ設定を追加する


### この工程でしていること
- Apache 経由で Tomcat にアクセスさせる
- URL を分かりやすくする
- セキュリティを向上させる


### 実行コマンド
```bash
vi /etc/httpd/conf/httpd.conf
```

### 追記内容
```bash
ProxyRequests Off
ProxyPass        /knowledge  http://127.0.0.1:8080/knowledge
ProxyPassReverse /knowledge  http://127.0.0.1:8080/knowledge
```


### 設定の意味

| 項目 | 内容 |
|------|------|
| ProxyRequests Off | フォワードプロキシ無効 |
| ProxyPass | Tomcat転送 |
| ProxyPassReverse | 応答変換 |


### 注意点（重要）
- mod_proxy が有効である必要がある
- 記述ミスは Apache 起動失敗の原因になる


### 確認方法とOKの目安
```bash
httpd -t
```
- Syntax OK が表示される


## 3-3. Apache を起動・自動起動設定する


### この工程でしていること
- Apache を常駐化する


### 実行コマンド
```bash
systemctl enable httpd
```
```bash
systemctl start httpd
```

### コマンドの意味
- enable：自動起動
- start：起動


### 確認方法とOKの目安
```bash
systemctl status httpd
```
- active (running)


## 3-4. アプリケーション用ディレクトリを作成する


### この工程でしていること
- Web アプリを配置する領域を準備する


### 実行コマンド
```bash
mkdir /usr/local/tomcat/webapps/knowledge
```

### 確認方法とOKの目安
```bash
ls /usr/local/tomcat/webapps
```
- knowledge が存在

## 3-5. アプリケーションを配置する


### この工程でしていること
- knowledge アプリを取得し配置する


### 実行コマンド
```bash
cd /usr/local/tomcat/webapps/knowledge
```
```bash
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
```

### 確認方法とOKの目安
```bash
ls
```
- knowledge.war が存在


## 3-6. WAR ファイルを展開する


### この工程でしていること
- ダウンロードしたファイルの解凍
- 不要なファイルの削除
- Java Web アプリを利用可能状態にする

### 実行コマンド
```bash
jar xf knowledge.war
```
```bash
rm knowledge.war
```
```bash
cd ../
```
```bash
chown tomcat:tomcat knowledge
```


### コマンドの意味
- chown
  - 所有者変更


### 確認方法とOKの目安
```bash
ls -ld knowledge
```
- tomcat tomcat


## 3-7. アプリケーションを反映させる


### この工程でしていること
- Tomcat に新しいアプリを認識させる


### 実行コマンド（必要時）
```bash
systemctl restart tomcat
```


### 確認方法とOKの目安
```bash
systemctl status tomcat
```
- active (running)


## 3-8. 動作確認（Tomcat 直接アクセス）


### この工程でしていること
- Tomcat 側で正常に動作しているか確認する


### 確認方法（ブラウザ）
`http://グローバルIP:8080/knowledge`

### OKの目安
- knowledge 初期画面が表示される
- エラー画面が出ない


## 3-9. 動作確認（Apache 経由アクセス：推奨）

### この工程でしていること
- Apache → Tomcat 連携が成功しているか確認する

### 確認方法（ブラウザ）
`http://グローバルIP/knowledge`

### OKの目安
- 同じ画面が表示される
- 8080 を指定せず閲覧可能

## 3-10. トラブル時の確認ポイント

### 表示されない場合

| 項目 | 確認 |
|------|------|
| Tomcat | systemctl status tomcat |
| Apache | systemctl status httpd |
| Proxy | httpd.conf |
| ポート | netstat -ln |
| ログ | catalina.out / error_log |

### 403 / 404 の場合

- 権限ミス
- ディレクトリ名誤り
- WAR 展開失敗

## 3-11. ここまでで完了していること

- Apache → Tomcat 連携完了
- Java アプリ配備完了
- Proxy 設定完了
- 外部公開完了
- AP サーバ運用可能

## ポイントまとめ

- ProxyPass は中継設定
- 127.0.0.1 は内部通信専用
- WAR は Web アプリ形式
- 所有者設定は必須
- 8080 直アクセスは切り分け用

