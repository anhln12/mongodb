Bộ lệnh kiểm tra định kỳ
|Mục đích|Lệnh|
|---|---|
|Trạng thái Replica Set|rs.status()|
|Kiểm tra Primary|db.hello()|
|Kiểm tra cấu hình|rs.conf()|
|Replication lag|rs.printSecondaryReplicationInfo()|
|Kiểm tra kết nối|db.adminCommand({ping:1})|
|Kiểm tra kết nối client|db.adminCommand({connectionStatus:1})|

Kiểm tra tài nguyên Linux:
```
iostat -xz 1 5
```
