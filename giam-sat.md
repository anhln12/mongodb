Tạo tài khoản giám sát

```
use admin

db.createUser({
  user: "mongoMonitor",
  pwd: passwordPrompt(),
  roles: [
    { role: "clusterMonitor", db: "admin" }
  ]
})
```

Script giám sát MongoDB  + Telegram
- MongoDB service bị dừng.
- Không có Primary.
- Một node trong Replica Set bị mất kết nối.
- Replication lag trên 60 giây.
- Disk usage trên 85%.
- RAM usage trên 90%.
- Gửi Telegram khi có sự cố và khi hệ thống phục hồi.

Tạo file:
```
install -d -m 700 /etc/mongodb
nano /etc/mongodb/telegram.env

# thêm nội dung
BOT_TOKEN="YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID="YOUR_TELEGRAM_CHAT_ID"

# Phân quyền
chmod 600 /etc/mongodb/telegram.env
```

Tạo script kiểm tra MongoDB
```
nano /etc/mongodb/healthcheck.js

# Nội dun
const admin = db.getSiblingDB("admin");

const authResult = admin.auth(
  "mongoMonitor",
  "CHANGE_MONITOR_PASSWORD"
);

if (authResult !== 1) {
  throw new Error("MongoDB authentication failed");
}

const result = admin.runCommand({
  replSetGetStatus: 1
});

if (result.ok !== 1) {
  throw new Error("Cannot get Replica Set status");
}

const self = result.members.find(m => m.self);
const primary = result.members.find(
  m => m.stateStr === "PRIMARY"
);

const peerDown = result.members.some(
  m => !m.self && m.health === 0
);

let lagSeconds = 0;

if (
  self &&
  primary &&
  self.optimeDate &&
  primary.optimeDate
) {
  lagSeconds = Math.max(
    0,
    Math.round(
      (primary.optimeDate - self.optimeDate) / 1000
    )
  );
}

let state = "OK";

if (!primary) {
  state = "NO_PRIMARY";
} else if (peerDown) {
  state = "PEER_DOWN";
} else if (
  !self ||
  !["PRIMARY", "SECONDARY"].includes(self.stateStr)
) {
  state = "MEMBER_ERROR";
} else if (lagSeconds > 60) {
  state = "REPLICATION_LAG";
}

print(JSON.stringify({
  state: state,
  member: self ? self.name : "UNKNOWN",
  role: self ? self.stateStr : "UNKNOWN",
  primary: primary ? primary.name : "NONE",
  lagSeconds: lagSeconds
}));
```

Bảo vệ file
```
chown root:root /etc/mongodb/healthcheck.js
chmod 600 /etc/mongodb/healthcheck.js
```

Script gửi Telegram và giám sát
```
nano /usr/local/sbin/mongodb-monitor.sh

# Nội dung
#!/bin/bash

set -u

source /etc/mongodb/telegram.env

STATE_DIR="/var/lib/mongodb-monitor"
STATE_FILE="${STATE_DIR}/last_state"

mkdir -p "$STATE_DIR"

send_telegram() {
    local MESSAGE="$1"

    curl -fsS --max-time 10 \
      -X POST \
      "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
      --data-urlencode "chat_id=${CHAT_ID}" \
      --data-urlencode "text=${MESSAGE}" \
      >/dev/null
}

HOST=$(hostname)

if ! systemctl is-active --quiet mongod; then
    CURRENT_STATE="MONGOD_DOWN"
    DETAIL="MongoDB service is not running"
else
    OUTPUT=$(
      mongosh \
        --host 127.0.0.1 \
        --tls \
        --tlsCAFile /etc/ssl/mongodb/ca.crt \
        --quiet \
        --file /etc/mongodb/healthcheck.js \
        2>&1
    ) || OUTPUT=""

    CURRENT_STATE=$(
      echo "$OUTPUT" | jq -r '.state // "CHECK_FAILED"' 2>/dev/null
    )

    ROLE=$(
      echo "$OUTPUT" | jq -r '.role // "UNKNOWN"' 2>/dev/null
    )

    LAG=$(
      echo "$OUTPUT" | jq -r '.lagSeconds // 0' 2>/dev/null
    )

    DETAIL="Role: ${ROLE}; Replication lag: ${LAG}s"
fi

DISK=$(
  df -P /var/lib/mongodb |
  awk 'NR==2 {gsub("%","",$5); print $5}'
)

MEMORY=$(
  free | awk '/Mem:/ {printf "%d", $3/$2*100}'
)

if [[ "$CURRENT_STATE" == "OK" && "$DISK" -ge 85 ]]; then
    CURRENT_STATE="DISK_USAGE_HIGH"
    DETAIL="MongoDB disk usage: ${DISK}%"
fi

if [[ "$CURRENT_STATE" == "OK" && "$MEMORY" -ge 90 ]]; then
    CURRENT_STATE="MEMORY_USAGE_HIGH"
    DETAIL="Memory usage: ${MEMORY}%"
fi

PREVIOUS_STATE="UNKNOWN"

if [[ -f "$STATE_FILE" ]]; then
    PREVIOUS_STATE=$(cat "$STATE_FILE")
fi

if [[ "$CURRENT_STATE" != "$PREVIOUS_STATE" ]]; then

    if [[ "$CURRENT_STATE" == "OK" ]]; then
        send_telegram \
          "RECOVERY: MongoDB ${HOST} is healthy again."
    else
        send_telegram \
          "ALERT: MongoDB ${HOST}
State: ${CURRENT_STATE}
Detail: ${DETAIL}"
    fi

    echo "$CURRENT_STATE" > "$STATE_FILE"
fi
```

Phân quyền
```
chmod 750 /usr/local/sbin/mongodb-monitor.sh
```

chmod 750 /usr/local/sbin/mongodb-monitor.sh


Lưu ý: Script này gửi cảnh báo khi trạng thái thay đổi, tránh gửi Telegram liên tục mỗi phút khi một lỗi chưa được khắc phục.

Chạy thử:
```
/usr/local/sbin/mongodb-monitor.sh
```

Thiết lập Cron
```
nano /etc/cron.d/mongodb-monitor
* * * * * root /usr/local/sbin/mongodb-monitor.sh
```










