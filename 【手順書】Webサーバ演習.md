# Apache 演習 手順書（配信設定・DocumentRoot変更・2台リダイレクト）

---

## 概要

本手順書では、以下の3つの構成を構築する。

1. Apache により /yakiniku/harami.html を公開する
2. DocumentRoot を /home/saba に変更し /miso/hoge.html を公開する
3. EC2 2台構成で /sa-mon.html を別サーバへリダイレクトする

---

## 前提条件

- Amazon Linux 系 OS を使用していること
- EC2 に Public IP が割り当てられていること
- セキュリティグループで 80/TCP が開放されていること
- root 権限で作業できること
- Apache がインストールされていること

---

# 【構成①】/yakiniku/harami.html を公開する

---

## 1. Apache をインストール・起動する

### この工程でしていること
- Web サーバを導入し、HTTP 通信を可能にする

### 実行コマンド
```bash
dnf install httpd -y
```
```bash
systemctl start httpd
```
```bash
systemctl enable httpd
```

### 確認方法とOKの目安
```bash
systemctl status httpd
```
```bash
netstat -ln | grep 80
```

- active (running)
- 80番ポート LISTEN

---

## 2. コンテンツ用ディレクトリを作成する

### この工程でしていること
- Web 公開用フォルダを作成する

### 実行コマンド
```bash
mkdir -p /var/www/html/yakiniku
```

### 確認方法
```bash
ls /var/www/html
```

- yakiniku が存在

---

## 3. HTML ファイルを作成する

### この工程でしていること
- 表示用コンテンツを配置する

### 実行コマンド
```bash
vi /var/www/html/yakiniku/harami.html
```

### 記載例
`Harami Page`

### 確認方法
ブラウザで確認　

`http://グローバルIP/yakiniku/harami.html`

- ページが表示される


---
# 【構成②】DocumentRoot を /home/saba に変更する 手順書

## 概要

本構成では、Apache の DocumentRoot を標準の /var/www/html から  
/home/saba に変更し、/miso/hoge.html を公開する。

---

## 1. 公開用ディレクトリを作成する

### この工程でしていること
- 新しい Web 公開用ルートディレクトリを作成する

### 実行コマンド
```bash
mkdir -p /home/saba/miso
```

### コマンドの意味
- mkdir : ディレクトリ作成
- -p    : 親ディレクトリも同時作成

### 確認方法とOKの目安
```bash
ls /home/saba
```

- miso ディレクトリが存在する

---

## 2. 公開用HTMLファイルを作成する

### この工程でしていること
- ブラウザ表示用のHTMLを作成する

### 実行コマンド
```bash
vi /home/saba/miso/hoge.html
```
### コマンドの意味
- vi：ファイル編集

### 確認方法とOKの目安
```bash
ls /home/saba/miso
```

- hoge.html が存在する

---

## 6. Apache 設定ファイルを編集する

### この工程でしていること
- DocumentRoot を変更する
- /home/saba へのアクセスを許可する

### 実行コマンド
```bash
cp /etc/httpd/conf/httpd.conf{,.back}
```
```bash
ls /etc/httpd/conf/
```
```bash
vi /etc/httpd/conf/httpd.conf
```

### 変更内容①（DocumentRoot）
```bash
DocumentRoot "/home/saba"
```
### 変更内容②（Directory 設定）
```bash
<Directory "/home/saba">
AllowOverride None
Require all granted
</Directory>
```
### 設定の意味
- DocumentRoot：公開ルートの指定
- Require all granted：全アクセス許可
- AllowOverride：.htaccess 制御可否

---

## 7. ディレクトリ権限を設定する

### この工程でしていること
- Apache が /home/saba 配下へアクセスできるようにする

### 実行コマンド
```bash
chmod 755 /home
```
```bash
chmod 755 /home/saba
```
```bash
chmod -R 755 /home/saba
```

### コマンドの意味
- chmod：権限変更
- 755：所有者のみ書込可、他は読取実行可
- -R：再帰的に適用

### 確認方法とOKの目安
```bash
ls -ld /home /home/saba /home/saba/miso
```

- drwxr-xr-x になっている

---

## 8. Apache を再起動する

### この工程でしていること
- 設定変更を反映する

### 実行コマンド
```bash
systemctl restart httpd
```

### コマンドの意味
- restart：停止→起動で再読み込み

### 確認方法とOKの目安
```bash
systemctl status httpd
```
- active (running)

---

## 9. 動作確認

### この工程でしていること
- DocumentRoot 変更後の配信確認を行う

### 確認URL
http://グローバルIP/miso/hoge.html

### OKの目安
- ページが表示される
- 403 / 404 / Timeout が出ない

---

# 【構成③】2台構成でリダイレクト設定する

---

## 10. EC2-2 に Apache を構築する

### この工程でしていること
- 転送先サーバを準備する

### 実行コマンド
```bash
dnf install httpd -y
```
```bash
systemctl start httpd
```
```bash
systemctl enable httpd
```

### 確認方法とOKの目安
```bash
systemctl status httpd
```
```bash
netstat -ln | grep 80
```

- active
- LISTEN 状態

---

## 11. 転送先コンテンツを配置する

### この工程でしていること
- リダイレクト先ファイルを準備する

### 実行コマンド
```bash
vi /var/www/html/aburi-sa-mon.html
```

### 確認方法とOKの目安
```bash
ls /var/www/html
```
- aburi-sa-mon.html が存在

---

## 12. EC2-1 にリダイレクト設定を追加する

### この工程でしていること
- 特定URLを別サーバへ転送する

### 実行コマンド
```bash
vi /etc/httpd/conf/httpd.conf
```

### 追記内容
```bash
Redirect 302 /sa-mon.html http://EC2-2のグローバルIP/aburi-sa-mon.html
```
### 設定の意味
- Redirect：転送設定
- 302：一時転送
- /sa-mon.html：元URL

---

## 13. Apache を再起動する

### 実行コマンド
```bash
systemctl restart httpd
```

### 確認方法とOKの目安
```bash
systemctl status httpd
```

- active

---

## 14. リダイレクト動作確認

### 確認URL
http://EC2-1のグローバルIP/sa-mon.html

### OKの目安
- EC2-2 のURLへ遷移する
- ページが表示される

---

## トラブル対処

### 403 Forbidden

原因：
- Directory 設定不足
- 権限不足

確認：
```bash
ls -ld /home /home/saba
```

---

### タイムアウト

原因：
- SG 80未開放
- Apache停止

確認：
```bash
systemctl status httpd
```
```bash
netstat -ln | grep 80
```

---

### リダイレクト失敗

原因：
- 転送先80未開放
- Private IP指定

確認：
- URL確認
- SG確認

---

### 完了条件

- /miso/hoge.html が表示される
- /sa-mon.html が転送される
- Apache が正常稼働している

---

# 【構成④】EC2-1のみアクセス許可・EC2-2 を拒否する構成 手順・注意点まとめ

## 概要

本手順書では、Apache に対して IP アドレス制御を設定し、

- EC2-1（自分）：アクセス許可
- EC2-2（相手）：アクセス拒否

となるように構成する。

curl による確認で、以下の結果となることを目標とする。

- 自分 → 自分：200 OK
- 相手 → 自分：403 Forbidden

---

## 構成概要
```bash
[EC2-1（自分）]
Apache
PrivateIP：172.31.xx.xx
↑ 許可
[EC2-2（相手）]
PrivateIP：172.31.yy.yy
↓ 拒否

両EC2は同一VPC内に存在
```

---

## ネットワーク構成イメージ
```bash
[VPC 172.31.0.0/16]
┌─────────────┐ ┌─────────────┐
│ EC2-1       │ │ EC2-2       │
│ (Apache)    │ │ (Client)    │
│ Web公開 │◀─────│ curl送信     │
│ IP制御あり   │ │             │
└─────────────┘ └─────────────┘
```

- 通信は Private IP を使用
- VPC 内通信のため IGW は不要
- 制御は Apache 側で実施

---

## セキュリティ設計

## セキュリティグループ設定（共通）

### Inbound（EC2-1）

| Port | Source | 用途 |
|------|--------|------|
| 80 | 172.31.0.0/16 | VPC内通信 |
| 22 | 自分IP | SSH |

※ 80 を VPC 内限定にすることで外部遮断

---

### Inbound（EC2-2）

| Port | Source | 用途 |
|------|--------|------|
| 22 | 自分IP | SSH |

※ Web公開不要

---

## セキュリティ設計のポイント

- AWS側では「VPC単位」で許可
- 個別拒否は Apache で制御
- SGでは相手を完全遮断しない（検証用）

---

## 1. Apache が稼働していることを確認

---

### この工程でしていること
- Web サービスが動作しているか確認する

### 実行コマンド（EC2-1）
```bash
systemctl status httpd
```
```bash
netstat -ln | grep 80
```

### OKの目安
- active (running)
- 80 LISTEN

---

## 2. 制御対象ディレクトリの作成・確認

---

### この工程でしていること
- 制御対象ディレクトリを作成する

### 実行コマンド

```bash
mkdir /var/www/html/apple/
```
```bash
vi /var/www/html/apple/ringo
```

### 確認コマンド
```bash
ls /var/www/html/apple
```
- コンテンツが存在

---

## 3. Apache アクセス制御設定

---

## 方法①：httpd.conf に直接設定（推奨）

---

### この工程でしていること
- IP 制御ルールを Apache に設定する

### 実行コマンド
```bash
vi /etc/httpd/conf/httpd.conf
```

---

### 追記設定（末尾）
```bash
<Directory "/var/www/html/apple">
Require ip 自分のPrivateIP
</Directory>
```
例：
Require ip 172.31.10.50

---

### 設定の意味

- Require ip ：指定IPのみ許可
- その他IPは自動拒否（403）

---

## 4. Apache 再起動

---

### この工程でしていること
- 設定反映

### 実行コマンド
```bash
systemctl restart httpd
```
---

### 確認
```bash
systemctl status httpd
```
---

## 5. 自分EC2からのアクセス確認

---

### この工程でしていること
- 許可IPから接続できるか確認

### 確認コマンド（EC2-1）
```bash
curl -LI http://自分のPrivateIP/apple/ringo
```
### OKの目安
HTTP/1.1 200 OK

---

## 6. 相手EC2からのアクセス確認

---

### この工程でしていること
- 拒否設定が動作しているか確認

### 確認コマンド（EC2-2）
```bash
curl -LI http://EC2-1のPrivateIP/apple/xxx
```

### OKの目安
HTTP/1.1 403 Forbidden

---

## 7. ログ確認（切り分け用）

---

### この工程でしていること
- 拒否ログを確認する

### 確認コマンド
```bash
tail /var/log/httpd/access_log
```
```bash
tail /var/log/httpd/error_log
```

---

### OKの目安
- 403 記録あり
- denied / forbidden 表示

---

## 8. よくある失敗例（試験頻出）

---

### ① 全部403になる

原因：
- Require ip のIP間違い
- Private IP変更後に未修正

---

### ② 相手も200になる

原因：
- Directory設定場所ミス
- Allow from all が残っている
- .htaccess優先

---

### ③ タイムアウト

原因：
- SG 80 未許可
- Apache 停止

---

## 9. AWSだけで制御する方法（参考）

---

### 方法：Security Group 制御

EC2-1 Inbound

| Port | Source |
|------|--------|
| 80 | 自分EC2のSG |

※ IP制御不要

制限は可能だが Apache制御の理解が弱くなるため研修では非推奨

---

## 10. まとめ

- VPC内通信 → Private IP
- IP制御 → Require ip
- 拒否時 → 403
- SG → 大枠制御
- Apache → 詳細制御

---

## 完了状態

- 自分：200 OK
- 相手：403 Forbidden
- ログで拒否確認可能
- AWS + Apache 両面防御