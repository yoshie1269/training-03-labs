下記2つの要件を満たすシェルを作成

- backup.listファイルに記載されたファイル名をバックアップしてくれる仕組み
- バックアップしたファイルがちゃんと存在するかチェックする仕組み

※引数と対話形式は今回使わない。シェルを引数無しで実行し、上記が実行されること。

```bash
#!/bin/bash

LIST_FILE="/home/ec2-user/onishi/backup.list"
BK_DIR="/home/ec2-user/onishi/backup"

if [ ! -f ${LIST_FILE} ]; then
        echo "backup.list がありません"
        exit 1
fi

mkdir -p ${BK_DIR}

while read SRC_FILE
do
        if [ -z ${SRC_FILE} ]; then
                continue
        fi

        if [ ! -f ${SRC_FILE} ]; then
                echo "エラー：バックアップ元ファイルが存在しません -> ${SRC_FILE}"
                continue
        fi

        NOW=$(date +"%Y%m%d%H%M%S")

        FILE_NAME=$(basename "${SRC_FILE}")
        BK_FILE="${BK_DIR}/${FILE_NAME}.${NOW}.bak"

        cp "${SRC_FILE}" "${BK_FILE}"

        if [ -f "${BK_FILE}" ]; then
                echo "OK: ${SRC_FILE} をバックアップしました -> ${BK_FILE}"
        else
                echo "NG: ${SRC_FILE} のバックアップに失敗しました"
        fi

done < "${LIST_FILE}"
```