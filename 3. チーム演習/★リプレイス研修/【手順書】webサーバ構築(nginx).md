# Webサーバ構築手順（nginx + Tomcat + Let's Encrypt）

本手順では以下を構築する。

- nginx Webサーバ
- Tomcat Reverse Proxy
- AP冗長化
- HTTPS通信（Let's Encrypt）
- SSL証明書更新

---

# 1 nginxインストール

## 1-1 nginxインストール

### コマンド
```bash
dnf install nginx -y
```

## 1-2 インストール確認

### コマンド
```bash
rpm -qa | grep nginx
```


## 1-3 バージョン確認

### コマンド
```bash
nginx -v
```

### 確認例
nginx version: nginx/1.28.2


## 1-4 nginx起動とステータス確認

### コマンド
```bash
systemctl status nginx.service
```

```bash
systemctl start nginx.service
```

```bash
systemctl status nginx.service
```


### 確認例
active (running)


## 1-5 自動起動設定と自動起動設定確認

### コマンド
```bash
systemctl enable nginx.service
```

```bash
systemctl is-enabled nginx.service
```

### 確認例
enabled


## 1-6 動作確認

ブラウザでアクセス
```bash
http://EC2のIP
```

### OKの状態

nginx welcomeページが表示される

---

# 2 nginx設定（Tomcat Proxy + 冗長化 + HTTPS前提）

## 2-1 nginx.confバックアップ

### コマンド
```bash
cp -a /etc/nginx/nginx.conf /etc/nginx/nginx.conf.bak
```


## 2-2 nginx設定編集

### 編集ファイル
```bash
vi /etc/nginx/nginx.conf
```


### server設定をコメントアウト

操作
```bash
Shift + V  
↓  
↓  
↓  
```
```bash
:s/^/# /
```

## 追記内容(必ずhttp{}の中に入れる)
```bash
server {

listen 80;  
server_name <名前解決したドメイン>;  
root /usr/share/nginx/html;  
index index.html;

include /etc/nginx/default.d/*.conf;

error_page 404 /404.html;

location = /404.html {  
}

error_page 500 502 503 504 /50x.html;

location = /50x.html {  
}

location = /health {  
add_header Content-Type text/plain;  
return 200 "ok\n";  
}

location / {

proxy_pass http://tomcat_backend;

proxy_http_version 1.1;  
proxy_set_header Connection "";

proxy_set_header Host              $host;  
proxy_set_header X-Real-IP         $remote_addr;  
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;  
proxy_set_header X-Forwarded-Proto $scheme;

proxy_connect_timeout 5s;  
proxy_read_timeout    60s;  
proxy_send_timeout    60s;

proxy_next_upstream error timeout invalid_header http_500 http_502 http_503 http_504;  
proxy_next_upstream_tries 2;

}

}
```

## 設定の意味

### proxy_pass

nginx → Tomcatへ通信転送


### /health

ロードバランサや監視用のヘルスチェック


### proxy_next_upstream

Tomcatがエラーの場合  
別サーバへ自動切替


# 2-3 Tomcat upstream設定

### 編集ファイル
```bash
vi /etc/nginx/conf.d/tomcat.conf
```


### 設定内容
```bash
upstream tomcat_backend {

server <AP1_PRIVATE_IP>:8080 max_fails=3 fail_timeout=10s;  
server <AP2_PRIVATE_IP>:8080 backup;  

keepalive 32;

}
```


### 設定の意味

- AP1：通常利用されるTomcat
- max_fails：3回失敗すると
- fail_timeout：10秒間切断
- backup：AP1が停止した場合のみ使用
- keepalive：接続再利用


# 2-4 nginx設定チェック
```bash
nginx -t
```


### OK例
syntax is ok  

test is successful


# 2-5 nginx設定反映
```bash
systemctl reload nginx
```


# 2-6 動作確認

ブラウザで確認
```bash
http://EC2のIP
```


### OKの状態

Tomcatページが表示される



# 3 Let's Encrypt 証明書発行

## 3-1 Certbotインストール
```bash
dnf install certbot python3-certbot-nginx
```


## 3-2 SSL証明書取得
```bash
certbot --nginx -d www2.teame2.entrycl.net
```


## 入力
```bash
メールアドレス  
root@teame2.entrycl.net

利用規約  
y

EFFメール  
y
```


## HTTPSアクセス確認
```bash
https://<名前解決したドメイン>
```


### OKの状態

HTTPS通信で

nginx → Tomcat

へ通信される

---

# 4 SSL証明書更新（cronで実行）

Let's Encrypt証明書は

90日有効


## 4-1 現在の証明書期限確認
```bash
openssl x509 -in /etc/letsencrypt/live/www2.teame2.entrycl.net/fullchain.pem -noout -dates
```


### 確認例
```bash
notBefore=Mar 5 00:00:00 2026 GMT  
notAfter=Jun 3 00:00:00 2026 GMT  
```

## 4-2 証明書強制更新
```bash
certbot renew --force-renewal
```


### 注意

通常の更新（certbot renew）

では期限が近い証明書のみ更新


## 4-3 nginxへ証明書反映
```bash
systemctl reload nginx
```


## 4-4 更新後の期限確認
```bash
openssl x509 -in /etc/letsencrypt/live/www2.teame2.entrycl.net/fullchain.pem -noout -dates
```


### 成功確認

notAfter の日付が更新されていれば成功


# 最終構成
```bash
Client  
↓  
HTTPS  
↓  
nginx (Reverse Proxy)  
↓  
HTTP  
↓  
Tomcat (AP Server)
```

- nginx Webサーバ
- Reverse Proxy
- Tomcat冗長化
- HTTPS通信
- Let's Encrypt証明書
- SSL自動更新