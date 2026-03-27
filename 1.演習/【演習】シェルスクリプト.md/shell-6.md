下記コマンドを使って 3～４サイトを監視するスクリプトを作成
curl -LI 監視したいURL -o /dev/null -w '%{http_code}\n' -s

- ifとforを使う
- 監視するサイトはsite.listというファイルを使ってシェルから読み込む
- 監視結果があとで見られるようにログを出力する機能をつける

```bash
#!/bin/bash

LIST_FILE="/home/ec2-user/onishi/site.list"
LOG_DIR="/home/ec2-user/onishi/log"
LOG_FILE="${LOG_DIR}/site_check.log"

mkdir -p "${LOG_DIR}"

if [ ! -f "${LIST_FILE}" ]; then
  echo "エラー: ${LIST_FILE} が存在しません" | tee -a "${LOG_FILE}"
  exit 1
fi

echo "===== $(date '+%Y-%m-%d %H:%M:%S') 監視開始 =====" >> "${LOG_FILE}"

for SITE in $(cat "${LIST_FILE}")
do
  HTTP_CODE=$(curl -LI "${SITE}" -o /dev/null -w '%{http_code}\n' -s)

  if [ "${HTTP_CODE}" = "200" ]; then
    echo "$(date '+%Y-%m-%d %H:%M:%S') OK ${SITE} HTTP_STATUS=${HTTP_CODE}" >> "${LOG_FILE}"
  else
    echo "$(date '+%Y-%m-%d %H:%M:%S') NG ${SITE} HTTP_STATUS=${HTTP_CODE}" >> "${LOG_FILE}"
  fi
done

echo "===== $(date '+%Y-%m-%d %H:%M:%S') 監視終了 =====" >> "${LOG_FILE}"
```