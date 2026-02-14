# Web-02 担当 詳細設計・構築手順書

担当者：[ ]
サーバ名：web02.wp.local

---

## 1. 役割
- 外部DNSバックアップ
- Postfix冗長

---

## 2. BIND構築
（Web-01と同様）

---

## 3. Postfix構築
（Web-01と同様）

---

## 4. 同期方針
- zoneファイル共有：[rsync/scp/TBD]
- 設定反映手順：[ ]

---

## 5. 確認
dig teamX.entrycl.net @web02

---

## 6. 障害対応
- Web01停止時の切替：[ ]