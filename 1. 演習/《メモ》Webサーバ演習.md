## 演習①
### Apacheを構築して /yakiniku/harami.html を見る
```bash
URL: http://グローバルIP/yakiniku/harami.html
```

■ 何をしているか
- ApacheのDocumentRoot（通常 /var/www/html）配下に
  /yakiniku/harami.html を置いて配信する

■ 注意点
- /var/www/html/yakiniku/harami.html のパスが正しいか
- 権限：ディレクトリに “実行(x)” がないと 403 になりやすい
- Apache起動してない・80待受してないとタイムアウト
- SGの80が閉じているとタイムアウト

■ 確認の観点（OKの目安）
- ローカル：サーバ自身で該当ファイルが存在して読める
- Apache：起動している
- ポート：80がLISTEN
- 外部：ブラウザで 200 OK が返る

## 演習②
### DocumentRootを /home/saba に変更して /miso/hoge.html を見る
```bash
URL: http://グローバルIP/miso/hoge.html
DocumentRoot: /home/saba
```

■ 何をしているか
- Apache が配信する「基準フォルダ(DocumentRoot)」を /home/saba に変える
- /home/saba/miso/hoge.html を公開する

■ 注意点（超重要：403・起動失敗の温床）
1) Apacheの設定変更は “DocumentRootだけ” では足りないことが多い
   - DocumentRootを変えるなら、対応する <Directory "..."> 設定も見直しが必要
   - 例：Require all granted / Options / AllowOverride など

2) /home 配下はデフォルトで Apache からアクセス禁止になっているケースがある
   - <Directory /> は “全拒否” が基本
   - 明示的に /home/saba を許可しないと 403 になる

3) パーミッション
   - /home, /home/saba, /home/saba/miso の各階層で “ディレクトリにx権限” が必要
   - 親ディレクトリの権限不足でも 403 になる

4) SELinux（有効環境）
   - /home をWeb公開にするとコンテキストで弾かれて 403 になることがある
   - 試験では「なぜ403？」の典型原因として出やすい

■ 試験で問われやすい “通信できない理由”
- SGで80が閉じている（タイムアウト）
- Apacheが80で待受してない（タイムアウト）
- DocumentRoot変更したが Directory許可してない（403）
- パスが違う（404）

## 演習③
### 2台構成：/sa-mon.html にアクセスすると 2台目の aburi-sa-mon.html にリダイレクト
```bash
URL: http://グローバルIP/sa-mon.html （EC2-1）
→ 2台目の http://EC2-2のグローバルIP/aburi-sa-mon.html にリダイレクト
```

■ 何をしているか
- EC2-1は /sa-mon.html を “配信する” のではなく
  HTTPのリダイレクト(3xx)で、EC2-2のURLへ転送する
- ブラウザはLocationヘッダを見てEC2-2へ再アクセスする

■ 注意点
1) EC2-2も外部から見える必要がある
   - EC2-2のSGで80が開いていないと、リダイレクト後にタイムアウトする
   - 「EC2-1は開くのに、転送先で詰まる」パターンが多い

2) “どの種類のリダイレクトか” を意識
   - 301（恒久）: ブラウザにキャッシュされやすい → 設定ミスが直りにくい
   - 302（暫定）: 演習・検証向き（キャッシュ影響が少ない）
   → 演習ではまず 302 が安全

3) URLの書き方
   - 相対パスではなく “完全なURL（http://IP2/...）” にするのが確実
   - Private IP を書くと、外部ブラウザは到達できず失敗する（必ずPublic側）

4) 2台目は “aburi-sa-mon.html” が正しい場所にあること
   - DocumentRoot配下に置く（/var/www/html/aburi-sa-mon.html など）

■ OKの目安
- EC2-1へアクセスすると、ブラウザのアドレスバーがEC2-2のURLに変わる
- HTTPステータスが 301/302 で Location: http://EC2-2... が返る
- 最終的にEC2-2のページが 200 OK で表示される


## セキュリティゲート（最重要チェック）

■ セキュリティグループ（最低限）
- EC2-1 / EC2-2 共通
  - 80/TCP : 0.0.0.0/0（演習でWeb公開するため）
  - 22/TCP : 自分のIPのみ（絶対に 0.0.0.0/0 にしない）
  - 443/TCP: 今回不要（https化しないなら）

■ サブネット/ルート（到達性）
- 「グローバルIPで見える」＝ Public Subnet + IGW経路が必要
- もしタイムアウトなら、SGより先に以下も疑う
  - Public IP / Elastic IP が付いているか
  - ルートテーブルに 0.0.0.0/0 → IGW があるか
  - NACLが遮断してないか（基本は許可でOK）

■ OS側（落とし穴）
- firewalld が有効なら 80/110/などOS firewallでも許可が必要なことがある
- SELinux が有効な環境だと DocumentRoot を /home にすると 403 になりやすい
  （Amazon Linux 2023 は SELinux 無効なことも多いが、環境次第で要注意）


【全体の構成イメージ（1枚図）】
```bash
(Internet / Browser)
        |
        |  HTTP 80
        v
+-------------------+               +-------------------+
| EC2-1 (Web1)      |  ③ Redirect   | EC2-2 (Web2)      |
| Apache            |-------------->| Apache            |
| ① /yakiniku/...   |  Location:    |  /aburi-sa-mon... |
| ② DocumentRoot変更 |  http://IP2/ |                   |
+-------------------+               +-------------------+
```

ポイント：
- ③は「EC2-1でコンテンツを返す」ではなく「EC2-2のURLへ転送(HTTP 3xx)」する
- ブラウザが最終的にアクセスするのはEC2-2（＝EC2-2も80を開ける必要がある）


