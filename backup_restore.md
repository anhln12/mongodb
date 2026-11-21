# Backup
```
date=`date +"%Y%m%d"`
#mongodump --host 192.168.68.xx --port 27017 --db xxx_portal --username xx --password xxx --authenticationDatabase admin --out /opt/backup-mongo-dbhome/
mongodump --host 192.168.68.xx --port 27017 --db xxx_portal --username xxx --password xxx --authenticationDatabase admin  --gzip --archive=/opt/backup-mongo-dbhome/xxx_portal_$date.gz
sleep 10
mongodump --host 192.168.68.xx --port 27017 --db xxx_multibank --username xxx --password xxx --authenticationDatabase admin  --gzip --archive=/opt/backup-mongo-dbhome/home_xxx_$date.gz
sleep 10
find /opt/backup-mongo-dbhome/ -name '*.gz' -mtime +3 -exec rm -f {} \;
```

# Restore
mongorestore --host localhost --port 27017 -u xxx_dev -p xxxx -d home_multibank --gzip --archive=/home/anhle4/home_multibank_20230616.gz --authenticationDatabase=admin


https://topdev.vn/blog/huong-dan-sao-luu-khoi-phuc-data-mongo-mongodump-mongorestore/



##2026##

Trên máy backup
```
apt update
apt install -y mongodb-database-tools curl jq gzip coreutils
```

Tạo cấu hình backup
```
mkdir -p /etc/mongodb
chmod 700 /etc/mongodb

nano /etc/mongodb/backup.env

# Nội dung
MONGO_HOSTS="mongo01:27017,mongo02:27017"
MONGO_REPLICA_SET="rs0"

MONGO_USER="mongoBackup"
MONGO_PASSWORD="YOUR_STRONG_PASSWORD"
MONGO_AUTH_DB="admin"

BACKUP_DIR="/backup/mongodb"
RETENTION_DAYS=14

# Telegram
BOT_TOKEN="YOUR_TELEGRAM_BOT_TOKEN"
CHAT_ID="YOUR_TELEGRAM_CHAT_ID"

# Backup server ngoài, để trống nếu chưa có
REMOTE_DEST=""
```

Phân quyền:
```
chmod 600 /etc/mongodb/backup.env
chown root:root /etc/mongodb/backup.env
```

Backup hoàn chỉnh
```
nano /usr/local/sbin/mongodb-backup.sh


#!/bin/bash

set -Eeuo pipefail
umask 077

CONFIG="/etc/mongodb/backup.env"

if [[ ! -f "$CONFIG" ]]; then
    echo "Missing config: $CONFIG"
    exit 1
fi

source "$CONFIG"

BACKUP_DIR="${BACKUP_DIR:-/backup/mongodb}"
RETENTION_DAYS="${RETENTION_DAYS:-14}"

TIMESTAMP=$(date '+%Y%m%d_%H%M%S')
HOSTNAME=$(hostname)

ARCHIVE="${BACKUP_DIR}/rs0-${TIMESTAMP}.archive.gz"
TEMP="${ARCHIVE}.tmp"

LOCK_FILE="/run/lock/mongodb-backup.lock"

mkdir -p "$BACKUP_DIR"
mkdir -p /run/lock

# Prevent concurrent backup
exec 9>"$LOCK_FILE"

if ! flock -n 9; then
    echo "Another backup process is running"
    exit 1
fi

send_telegram() {
    local MESSAGE="$1"

    if [[ -z "${BOT_TOKEN:-}" || -z "${CHAT_ID:-}" ]]; then
        return 0
    fi

    curl -fsS --max-time 15 \
        -X POST \
        "https://api.telegram.org/bot${BOT_TOKEN}/sendMessage" \
        --data-urlencode "chat_id=${CHAT_ID}" \
        --data-urlencode "text=${MESSAGE}" \
        >/dev/null || true
}

cleanup() {
    local RC=$?

    if [[ "$RC" -ne 0 ]]; then
        rm -f "$TEMP"

        send_telegram "MONGODB BACKUP FAILED
Host: ${HOSTNAME}
Time: $(date)
Exit code: ${RC}"
    fi
}

trap cleanup EXIT

# Validate required configuration
for VAR in MONGO_HOSTS MONGO_REPLICA_SET MONGO_USER \
           MONGO_PASSWORD MONGO_AUTH_DB; do
    if [[ -z "${!VAR:-}" ]]; then
        echo "Missing variable: $VAR"
        exit 1
    fi
done

# Check disk usage
DISK_USAGE=$(df -P "$BACKUP_DIR" |
    awk 'NR==2 {gsub("%","",$5); print $5}')

if [[ "$DISK_USAGE" -ge 85 ]]; then
    echo "Backup disk usage exceeds 85%"
    exit 1
fi

# Build MongoDB URI without embedding password
URI="mongodb://${MONGO_HOSTS}/?replicaSet=${MONGO_REPLICA_SET}&authSource=${MONGO_AUTH_DB}&tls=true"

echo "Starting MongoDB backup..."
echo "Time: $(date)"
echo "Destination: $ARCHIVE"

# Full replica set dump with oplog
mongodump \
    --uri="$URI" \
    --username="$MONGO_USER" \
    --password="$MONGO_PASSWORD" \
    --authenticationDatabase="$MONGO_AUTH_DB" \
    --gzip \
    --oplog \
    --archive="$TEMP"

# Validate output
if [[ ! -s "$TEMP" ]]; then
    echo "Backup archive is empty"
    exit 1
fi

# Atomic rename
mv "$TEMP" "$ARCHIVE"

# SHA256 checksum
sha256sum "$ARCHIVE" > "${ARCHIVE}.sha256"

# Verify checksum
sha256sum -c "${ARCHIVE}.sha256"

# Copy to remote backup server
if [[ -n "${REMOTE_DEST:-}" ]]; then
    rsync -av --partial \
        "$ARCHIVE" \
        "${ARCHIVE}.sha256" \
        "${REMOTE_DEST}/"
fi

# Remove old backups
find "$BACKUP_DIR" \
    -type f \
    \( -name 'rs0-*.archive.gz' -o -name 'rs0-*.archive.gz.sha256' \) \
    -mtime "+${RETENTION_DAYS}" \
    -delete

SIZE=$(du -h "$ARCHIVE" | awk '{print $1}')

send_telegram "MONGODB BACKUP SUCCESS
Host: ${HOSTNAME}
Replica Set: ${MONGO_REPLICA_SET}
File: $(basename "$ARCHIVE")
Size: ${SIZE}
Time: $(date)
Remote: ${REMOTE_DEST:-LOCAL ONLY}"

echo "Backup completed successfully"
echo "File: $ARCHIVE"
echo "Size: $SIZE"

```

Phân quyền
```
chmod 750 /usr/local/sbin/mongodb-backup.sh
chown root:root /usr/local/sbin/mongodb-backup.sh
```

Chạy thử nghiệm
```
/usr/local/sbin/mongodb-backup.sh
```

Kiểm tra checksum
```
sha256sum -c rs0-*.archive.gz.sha256
```

Cấu hình cron tự động
```
nano /etc/cron.d/mongodb-backup
30 1 * * * root /usr/local/sbin/mongodb-backup.sh >> /var/log/mongodb-backup.log 2>&1
```

Restore backup
```
mongorestore \
  --uri="mongodb://mongoAdmin@mongo01:27017/?authSource=admin&tls=true" \
  --password \
  --gzip \
  --archive=/backup/mongodb/rs0-YYYYMMDD_HHMMSS.archive.gz \
  --oplogReplay
```



