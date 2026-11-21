Với hệ thống của bạn đang dùng MongoDB 8.0, 2 node Replica Set, mỗi node 16 GB RAM và SSD 500 GB, tôi khuyên nên cấu hình rõ các nhóm sau:
- Kết nối: giới hạn số kết nối vào server, connection pool phía ứng dụng.
- Bộ nhớ: WiredTiger cache, giới hạn RAM dành cho cache.
- Đồng bộ: replication oplog, heartbeat, election.
- Hiệu năng: journal, slow query profiling, index
- Bảo vệ tài nguyên: giới hạn tiến trình, file descriptor và giám sát kết nối.

1. Các tham số tương đương với database khác
|MongoDB|Ý nghĩa|Giá trị mặc định / đề xuất|
|---|---|---|
|net.maxIncomingConnections|Số kết nối đồng thời tối đa vào MongoDB|Linux tự tính theo ulimit; có thể đặt rõ|
|maxPoolSize|Số connection tối đa trong pool của ứng dụng|100|
|minPoolSize|Số connection tối thiểu trong pool|0|
|maxConnecting|Số kết nối pool được tạo đồng thời|2|
|storage.wiredTiger.engineConfig.cacheSizeGB|RAM dành cho WiredTiger cache|Khoảng 7.5 GB với RAM 16 GB|
|operationProfiling.slowOpThresholdMs|Ngưỡng ghi nhận truy vấn chậm|Thường bắt đầu 100 ms|
|replication.oplogSizeMB|Dung lượng oplog phục vụ replication|MongoDB tự tính nếu không cấu hình|

Lưu ý: maxIncomingConnections là giới hạn phía database, còn maxPoolSize là giới hạn pool phía ứng dụng. Hai thông số này không giống nhau.

2. Cấu hình đề xuất cho mỗi node 16 GB RAM

File /etc/mongod.conf:

```
storage:
  dbPath: /var/lib/mongodb
  journal:
    enabled: true
  wiredTiger:
    engineConfig:
      cacheSizeGB: 7

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 127.0.0.1,mongo01
  maxIncomingConnections: 2000

processManagement:
  timeZoneInfo: /usr/share/zoneinfo

operationProfiling:
  mode: slowOp
  slowOpThresholdMs: 100
  slowOpSampleRate: 1.0

replication:
  replSetName: rs0

security:
  authorization: enabled
  # Giữ nguyên cấu hình clusterAuthMode và TLS
  # theo phương án xác thực Replica Set đang triển khai.
```

Giải thích các giá trị:
- cacheSizeGB: 7: dành 7 GB cho WiredTiger, chừa RAM cho filesystem cache, kết nối và tiến trình khác. MongoDB mặc định sử dụng khoảng 50% RAM trừ 1 GB cho WiredTiger cache.
- maxIncomingConnections: 2000: là mức giới hạn khởi đầu tham khảo, không có nghĩa MongoDB cần hoặc nên duy trì 2.000 kết nối. Phải kiểm tra ulimit trước khi áp dụng.
- slowOpThresholdMs: 100: giúp phát hiện truy vấn mất trên 100 ms. Không có nghĩa tất cả truy vấn phải dưới 100 ms.

3. Kiểm tra giới hạn kết nối trên Ubuntu
```
ulimit -n
systemctl show mongod -p LimitNOFILE
cat /proc/$(pidof mongod)/limits | grep -i "open files"
```

Kiểm tra kết nối thực tế:
```
db.serverStatus().connections
```

Kiểm tra WiredTiger cache:
```
db.serverStatus().wiredTiger.cache
```

Kiểm tra truy vấn chậm:
```
db.getProfilingStatus()
```

4. Cấu hình connection pool phía ứng dụng
```
mongodb://mongo01:27017,mongo02:27017/mydb?replicaSet=rs0&maxPoolSize=100&minPoolSize=10&maxConnecting=5
```

Không nên tăng maxPoolSize tùy tiện. Ví dụ có 10 application server, mỗi server tạo pool 100 kết nối đến mỗi node thì có thể phát sinh khoảng 1.000 kết nối đến mỗi node.
