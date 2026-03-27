# cron + systemd 手順書

## 1 cronuser 作成（rootで実施）

### この工程でしていること
cron実行専用ユーザーを作成する

### コマンド
```bash
sudo su -  
```
```bash
useradd cronuser 
```
```bash 
passwd cronuser  
```

### コマンドの意味
- useradd：ユーザー作成  
- passwd：パスワード設定  

### 確認方法
```bash
id cronuser  
```

### OKの目安
- uid が表示される  
- ユーザーが存在する  

---

## 2 cron（cronie）インストール

### この工程でしていること
cron機能（定期実行機能）を導入

### コマンド
```bash
dnf install -y cronie  
```
```bash
systemctl start crond  
```
```bash
systemctl enable crond  
```

### コマンドの意味
- cronie：cronサービス本体  
- start：起動  
- enable：自動起動  

### 確認方法
```bash
systemctl status crond  
```

### OKの目安
- active (running)  

---

## 3 cronuser に切り替え

### この工程でしていること
cron設定をユーザー単位で実施

### コマンド
```bash
su - cronuser  
```

### 確認方法
```bash
whoami  
```

### OKの目安
- cronuser と表示される  

---

## 4 crontab 設定

### この工程でしていること
定期実行処理を設定

### コマンド
```bash
crontab -e  
```
### 追記内容
```bash
0 15 * * 1-5 mkdir -p ~/memo_dir && echo "memo" > ~/memo_dir/memo.txt  
```

### コマンドの意味
- 0 15：15:00実行  
- 1-5：月〜金  
- mkdir -p：ディレクトリ作成（存在してもOK）  
- echo：文字出力  
- >：ファイル作成（上書き）  

### 確認方法
```bash
crontab -l  
```

### OKの目安
- 設定内容が表示される  

---

## 5 cron 動作確認

### この工程でしていること
cronが正しく実行されるか確認

### 確認方法
```bash
ls ~/memo_dir  
```
```bash
cat ~/memo_dir/memo.txt  
```

### OKの目安
- memo.txt が作成されている  
- 内容に "memo" がある  

---

## 6 systemd timer 設定（rootで実施）

### この工程でしていること
systemdによる定期実行設定

---

## 7 service 作成

### この工程でしていること
実行する処理を定義

### コマンド
```bash
vi /etc/systemd/system/hogehoge.service  
```

### 設定内容
```bash
[Unit]  
Description=Write hogehoge log  

[Service]  
Type=oneshot  
ExecStart=/bin/bash -c 'echo "hogehoge" >> /var/log/replace_cron.log'  
```

### コマンドの意味
- oneshot：1回だけ実行  
- ExecStart：実行コマンド  
- >>：追記  

### 確認方法
```bash
cat /etc/systemd/system/hogehoge.service  
```

### OKの目安
- 設定内容が正しく保存されている  

---

## 8 timer 作成

### この工程でしていること
実行スケジュールを定義

### コマンド
```bash
vi /etc/systemd/system/hogehoge.timer  
```

### 設定内容
```bash
[Unit]  
Description=Run every 30 minutes  

[Timer]  
OnCalendar=*:0/30  
Persistent=true  

[Install]  
WantedBy=timers.target  
```

### コマンドの意味
- OnCalendar：スケジュール指定  
- 0/30：30分ごと  
- Persistent：再起動後も実行  

### 確認方法
```bash
cat /etc/systemd/system/hogehoge.timer  
```

### OKの目安
- 設定が保存されている  

---

## 9 有効化・起動

### この工程でしていること
timerを有効化して動作開始

### コマンド
```bash
systemctl daemon-reexec  
```
```bash
systemctl start hogehoge.timer  
```
```bash
systemctl enable hogehoge.timer  
```

### コマンドの意味
- daemon-reexec：設定再読み込み  
- start：起動  
- enable：自動起動  

### 確認方法
```bash
systemctl status hogehoge.timer  
```

### OKの目安
- active (waiting)  

---

## 10 動作確認

### この工程でしていること
timerが実行されているか確認

### コマンド
```bash
systemctl list-timers  
```
```bash
cat /var/log/replace_cron.log  
```

### OKの目安
- タイマー一覧に表示される  
- ログに "hogehoge" が追記されている  

---

## 11 ポイントまとめ

### cron

- ユーザー単位で設定  
- 書式：分 時 日 月 曜日  

---

### systemd

- rootで設定  
- service + timer のセット  

---

### よくあるミス

- cronとsystemdを混同  
- 一般ユーザーでsystemd操作  
- >（上書き）と >>（追記）を間違える  