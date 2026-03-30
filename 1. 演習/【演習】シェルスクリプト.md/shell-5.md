CPUの使用率求めて、その結果をHTMLとして出力してwebサイトから見られるようにする

/var/www/html/onishi/index.html

```bash
#!/bin/bash

OUT_DIR="/var/www/html/onishi"
OUT_FILE="${OUT_DIR}/index.html"

mkdir -p "${OUT_DIR}"

CPU_IDLE=$(top -bn1 | grep "%Cpu(s)" | awk '{print $8}')
CPU_USED=$(awk "BEGIN {print 100 - ${CPU_IDLE}}")
DATE_NOW=$(date '+%Y-%m-%d %H:%M:%S')

cat <<EOF > "${OUT_FILE}"
<!DOCTYPE html>
<html lang="ja">
<head>
  <meta charset="UTF-8">
  <title>CPU使用率</title>
</head>
<body>
  <h1>CPU使用率</h1>
  <p>現在時刻: ${DATE_NOW}</p>
  <p>CPU使用率: ${CPU_USED}%</p>
</body>
</html>
EOF

echo "HTML出力完了: ${OUT_FILE}"
```

