# tomcat構築
## ①OpenJDK 17.0.9導入

### パッケージをインストール
```
sudo dnf install -y java-17-amazon-corretto-devel`
```
### javaのバージョン確認
```
java --version
```
### 実行結果
```
openjdk 17.0.9（以下略）
```
### javacのバージョン確認
```
javac --version
```
### 実行結果
```
javac 17.0.9
```
### javaファイル作成(やらなくてもOK)
```
ディレクトリ移動
cd /usr/bin
```
```
javaファイル作成
sudo vi HelloWorld.java
```
```
以下の記述を追加
public class HelloWorld {
	public static void main(String[] args) {
		System.out.println("java実行出来たよ！");
	}
}
```
### javaファイルコンパイル(やらなくてもOK)
```
javacでコンパイルし、javaファイルを実行可能状態にする
sudo javac HelloWorld.java
```
```
コンパイル出来ていることを確認
ls
```
```
実行結果
HelloWorld.class　HelloWorld.java （以下略）
```
### コンパイルしたクラスの実行(やらなくてもOK)
```
sudo java HelloWorld
「Java実行出来たよ！」と表示されれば成功
```
## ②Tomcat9.0.82導入
### tomcat専用ユーザーを作成する。
```
セキュリティの観点から、rootユーザーでtomcatを起動するのは危ないため
sudo useradd -r -s /sbin/nologin tomcat
```
### アカウントの確認
```
id tomcat
実行結果 tomcatグループのtomcatユーザーがちゃんとあるよって言ってる
uid=992(tomcat) gid=992(tomcat) groups=992(tomcat)
```
### アカウントの確認２
```
ログインは出来ないユーザーだよって言ってる
grep tomcat /etc/passwd
実行結果
tomcat:x:992:992::/home/tomcat:/sbin/nologin
```
### ホームディレクトリ移動
```
cd
```
### 公式からアーカイブをダウンロード
```
sudo wget https://archive.apache.org/dist/tomcat/tomcat-9/v9.0.82/bin/apache-tomcat-9.0.82.tar.gz 
```
### ダウンロードしたパッケージを展開し、一覧で表示する
```
sudo tar -zxvf apache-tomcat-9.0.82.tar.gz 
```
### ダウンロードしたapache-tomcat-9.0.82ファイルを、/usr/local/srcディレクトリに移動する
```
sudo mv apache-tomcat-9.0.82 /usr/local/src/
```
### シンボリックリンクを作成する（ショートカットみたいなもの）
```
sudo ln -s /usr/local/src/apache-tomcat-9.0.82/ /usr/local/src/tomcat
```
### 先ほど作成したtomcatユーザーに、/apache-tomcat-9.0.82、/tomcatの権限を付与
```
sudo chown -R tomcat:tomcat /usr/local/src/apache-tomcat-9.0.82
sudo chown -R tomcat:tomcat /usr/local/src/tomcat
```
### 権限が変更されたか確認
```
cd /usr/local/src
ls -l
実行結果（下記のような表示があれば成功）
drwxr-xr-x. 9 tomcat tomcat 16384  1月 11 02:37 apache-tomcat-9.0.82
lrwxrwxrwx. 1 tomcat tomcat    36  1月 11 02:31 tomcat -> /usr/local/src/apache-tomcat-9.0.82/
```

### ユニットファイルをtomcat.serviceと言う名前で、"/etc/systemd/system/"に作成し、エディタ起動
```
sudo vi /etc/systemd/system/tomcat.service
```
### 以下をエディタに記述
```
[Unit]
Description=Tomcat Web Server
After=network.target

[Service]
Type=forking
User=tomcat
Group=tomcat
Environment="CATALINA_BASE=/usr/local/src/tomcat"
Environment="CATALINA_HOME=/usr/local/src/tomcat"
Environment="CATALINA_PID=/usr/local/src/tomcat/temp/tomcat.pid"
ExecStart=/usr/local/src/tomcat/bin/startup.sh
ExecStop=/usr/local/src/tomcat/bin/shutdown.sh

[Install]
WantedBy=multi-user.target
```
### ユニットファイルの読み込み（systemdに変更を反映させる）
```
daemon=デーモンとは、常駐プログラムのこと。
sudo systemctl daemon-reload
```
```
ユニットファイルの有効化（自動起動設定を有効にする）
sudo systemctl enable tomcat.service
```

### エディタで .bash_profile ファイル開く
```
sudo vi ~/.bash_profile

エディタに以下の行を最終行に追加する
export CATALINA_HOME=/usr/local/src/tomcat

bash_profileシェルを実行する
source ~/.bash_profile
```

### tomcatの自動起動、有効化
```
sudo systemctl enable tomcat.service
```
### tomcatを起動
```
sudo systemctl start tomcat.service
```
### tomcatの状態確認
```
sudo systemctl status tomcat.service
実行結果
 tomcat.service - Tomcat Web Server
     Loaded: loaded (/etc/systemd/system/tomcat.service; enabled; preset: disabled)
     Active: active (running) since Wed 2024-01-17 19:45:26 JST; 4min 42s ago
```

### setenvを作成する
```
vi /usr/local/src/tomcat/bin/setenv.sh
---ここから--------------------------------------
#!/bin/sh
export CATALINA_HOME=/usr/local/src/tomcat
export JAVA_HOME=/usr/lib/jvm/java-17-amazon-corretto.x86_64
export JAVA_OPTS="-Xms128m -Xmx512m"
---ここまで--------------------------------------
```
### server.xmlを設定する
```
ここで設定する箇所は自動デプロイを無効にする
vi /usr/local/src/tomcat/conf/server.xml
unpackWARsとautoDeployをfalseにする
```
### Apacheのインストール
```
sudo dnf install httpd -y
```

### proxy設定を追記
```
vi /etc/httpd/conf/httpd.conf
一番下に下記を追記

---ここから--------------------------------------
ProxyRequests Off
ProxyPass /test http://127.0.0.1:8080/test
ProxyPassReverse /test http://127.0.0.1:8080/test
---ここまで--------------------------------------
```

### 自動有効化と起動
 ```
 systemctl enable httpd
 systemctl start httpd
```

### javaアプリの配備
```
配備場所のディレクトリ作成
mkdir /usr/local/src/tomcat/webapps/knowledge
cd /usr/local/src/tomcat/webapps/knowledge
```
### アプリケーションのダウンロード
```
wget https://github.com/support-project/knowledge/releases/download/v1.13.1/knowledge.war
```
### ダウンロードしたファイルの解凍
```
jar xf knowledge.war
```

### 不要ファイルの削除
```
rm knowledge.war
```

### アプリケーションの権限変更
```
cd ../
chown tomcat:tomcat knowledge
```

### 動作確認
```
-アクセス確認
http://グローバルIP:8080/knowledge 
        
-アクセス確認
http://グローバルIP/knowledge
でアクセスできることを確認する。
```
