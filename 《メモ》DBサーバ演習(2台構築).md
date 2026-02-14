# LAMP＋WordPress 完全構築フロー：全体の注意点（セキュリティ/運用）＋構成図

---

## 1. 全体の「落とし穴」チェックリスト

### ネットワーク（セキュリティグループ / 到達性）
- 80/TCP（HTTP）を開けないと WordPress セットアップ画面が表示されない
- 22/TCP（SSH）は「自分のIPのみ」に制限する（0.0.0.0/0 は避ける）
- DB を別EC2にする場合は「DB側の 3306/TCP を Web(WordPress)SG からのみ許可」にする
- 同一EC2で DB を動かす場合も、基本は 3306 を外部に開けない（ローカル利用に限定）

### OS/ミドルウェア（起動順・自動起動）
- Apache / MariaDB は「起動 + enable（自動起動）」までセット
- 設定変更後は再起動が必要
  - Apache 設定変更 → httpd restart
  - wp-config.php変更 → Webアクセスで即反映（サービス再起動不要なことが多いが、権限変更や PHP モジュール追加は Apache 再起動が必要になることがある）

### WordPress（設定ファイルとセキュリティキー）
- wp-config.php の DB_NAME / DB_USER / DB_PASSWORD が一致しないと「DB接続エラー」
- SALT（Authentication Unique Keys and Salts）を入れないとセッション保護が弱い
- WordPress 配置場所（/var/www/html 直下 or /blog/）を混在させると URL が分かりにくくなる（運用方針を決める）

### 権限
- /var/www 配下の所有者・権限が不適切だと…
  - 画面が 403 になる
  - 更新/画像アップロードが失敗する
  - プラグイン導入ができない
- 研修手順のように apache 所有にするなら、更新は基本 Web 経由で行う想定になる
- 「権限を緩めすぎる（例：777）」はNG（事故・改ざんリスク増）

### DB（ユーザー権限の基本）
- root は DB 管理専用（WordPress から使わない）
- WordPress 専用ユーザーを作成し、対象DBだけ権限付与（GRANT）
- パスワードは強力に、流用しない

---

## 2. セキュリティゲート

### セキュリティグループ例（1台構成：Web＋DB同居）
- Inbound
  - 22/TCP：自分のIPのみ（SSH）
  - 80/TCP：0.0.0.0/0（HTTP 公開）
- Outbound
  - デフォルト許可で可

※ DB（3306）は外部に開けない（同一EC2内利用なら不要）

### セキュリティグループ例（2台構成：Web(WordPress) と DB を分離）
- Web(WordPress)側 Inbound
  - 22/TCP：自分のIPのみ
  - 80：0.0.0.0/0
- DB側 Inbound
  - 22/TCP：自分のIPのみ
  - 3306/TCP：Web(WordPress)のセキュリティグループからのみ許可

---

## 3. 構成図（1枚図：LAMP＋WordPress）

### 1台構成（WordPress＋Apache＋PHP＋MariaDBが同居）
```bash
[Internet/Browser]
   |
   |  HTTP(80) 
   v
[EC2 (Public Subnet)]
   |
   |-- [Apache httpd]
   |       |
   |       |  PHP実行（php-fpm / phpモジュール）
   |       v
   |    [WordPress (PHP)]
   |       |
   |       |  DB接続（localhost）
   |       v
   |    [MariaDB]
   |
   +-- [Storage]
          /var/www/html (WordPress配置)
          /var/spool/mail 等
```
ポイント：
- ブラウザ → Apache → PHP(WordPress) → MariaDB の順で処理が流れる
- 80 の開放がないとブラウザから到達できない
- DBは外部公開しない（同居構成なら特に）

---

### 2台構成（DB分離）
```bash
[Internet/Browser]
   |
   |  HTTP(80) 
   v
[EC2-Web]
   |
   |-- [Apache + WordPress(PHP)]
   |
   |  DB接続（TCP/3306）
   v
[EC2-DB]
   |
   +-- [MariaDB]
```

ポイント：
- DBは Private Subnet に置く（外部から直接触れない）
- DBの3306は WebサーバのSGからのみ許可（最小権限）

---

## 4. 典型トラブルと原因

### 画面が開かない（タイムアウト）
- SGで 80/443 が閉じている
- サブネット/ルートテーブルが Public になっていない（IGW 経路なし）
- Apache が起動していない

### 403 Forbidden
- /var/www/html の権限・所有者が不適切
- Directory 設定や AllowOverride の設定ミス
- 配置場所と DocumentRoot のズレ

### WordPress の「DB接続確立エラー」
- MariaDB が起動していない
- wp-config.php の DB情報が不一致
- DBユーザー権限（GRANT）が不足
- DB分離構成で 3306 がSGで許可されていない

### 画像アップロードできない / 画像処理が失敗
- php-gd がない
- /var/www 配下の権限不足

