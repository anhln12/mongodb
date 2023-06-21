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
