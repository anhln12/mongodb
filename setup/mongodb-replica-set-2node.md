Ta triển khai MongoDB Replica Set gồm 2 Data Node, không sử dụng Arbiter.

Mô hình:
- Node 1: Primary – nhận ghi và đọc
- Node 2: Secondary – đồng bộ dữ liệu từ Node 1.
- Khi Node 1 chết: Node 2 không tự lên Primary. Bạn sẽ phải can thiệp thủ công

1. Cấu hình đề xuất

Sử dụng Ubuntu 22.04/24.04 và MongoDB 8.0.

|Thông số|Node 1|Node 2|
|---|---|---|
|Role ban đầu|Primary|Secondary|
|IP|10.10.10.11|10.10.10.12|
|Port|27017|27017|
|Replica Set|rs0|rs0|
|Data directory|/var/lib/mongodb|/var/lib/mongodb|

2. Cài MongoDB Community Edition trên cả 2 node
```
# Import the public key.
sudo apt-get install gnupg curl
curl -fsSL https://pgp.mongodb.com/server-8.0.asc | sudo gpg -o /usr/share/keyrings/mongodb-server-8.0.gpg --dearmor

# Create the list file
echo "deb [ arch=amd64,arm64 signed-by=/usr/share/keyrings/mongodb-server-8.0.gpg ] https://repo.mongodb.org/apt/ubuntu noble/mongodb-org/8.0 multiverse" | sudo tee /etc/apt/sources.list.d/mongodb-org-8.0.list

# Reload the package database
sudo apt-get update

# Install MongoDB Community Server.
sudo apt-get install -y mongodb-org
systemctl enable --now mongod

# Verify
mongod --version
```
3. Tạo KeyFile xác thực giữa 2 node Replica Set

Khi bật authentication trên Replica Set, các node mongod phải xác thực lẫn nhau bằng cùng một keyFile.

Tạo keyFile trên mongo1
```
sudo openssl rand -base64 756 | sudo tee /etc/mongod-keyfile > /dev/null
sudo chown mongodb:mongodb /etc/mongod-keyfile
sudo chmod 400 /etc/mongod-keyfile
```

Copy /etc/mongod-keyfile sang mongo2 và đảm bảo quyền giống hệt:
```
# Chạy từ mongo1 (hoặc copy qua máy trung gian)
scp /etc/mongod-keyfile root@10.0.0.12:/tmp/mongod-keyfile

# Trên mongo2
sudo mv /tmp/mongod-keyfile /etc/mongod-keyfile
sudo chown mongodb:mongodb /etc/mongod-keyfile
sudo chmod 400 /etc/mongod-keyfile
```

4. Cấu hình /etc/mongod.conf trên cả 2 node
 
Chỉnh sửa file /etc/mongod.conf trên cả mongo1 và mongo2 (chỉ đổi bindIp nếu bạn muốn bind riêng IP nội bộ thay vì 0.0.0.0):
```
storage:
  dbPath: /var/lib/mongodb

systemLog:
  destination: file
  logAppend: true
  path: /var/log/mongodb/mongod.log

net:
  port: 27017
  bindIp: 0.0.0.0

security:
  authorization: enabled
  keyFile: /etc/mongod-keyfile

replication:
  replSetName: rs0
```

Khởi động và enable service trên cả 2 node:
```
sudo systemctl daemon-reload
sudo systemctl enable mongod
sudo systemctl restart mongod
sudo systemctl status mongod --no-pager
```

5. Khởi tạo Replica Set và tạo user Admin

Kết nối vào mongo1 qua mongosh
```
mongosh --host mongo1 --port 27017
```

Khởi tạo Replica Set với 2 node:
```
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo1:27017" },
    { _id: 1, host: "mongo2:27017" }
  ]
})
```

Tạo user quản trị (root/userAdminAnyDatabase) trên admin
```
use admin
db.createUser({
  user: "dbadmin",
  pwd: "StrongPassword_Here_2026!",
  roles: [
    { role: "root", db: "admin" }
  ]
})
```

Kiểm tra trạng thái Replica Set và xác thực

```
rs.status()
```

Kiểm tra kết nối ngoài bằng URI có xác thực
```
mongosh "mongodb://dbadmin:StrongPassword_Here_2026!@mongo1:27017,mongo2:27017/admin?replicaSet=rs0&authSource=admin"
```

Tạo tài khoản riêng cho Application
```
use appdb

db.createUser({
  user: "appuser",
  pwd: passwordPrompt(),
  roles: [
    { role: "readWrite", db: "appdb" }
  ]
})
```
Kiểm tra kết nối ngoài bằng URI có xác thực
```
mongodb://appuser:PASSWORD@mongo01:27017,mongo02:27017/appdb?replicaSet=rs0&authSource=appdb
```

Tạo tài khoản backup riêng
```
use admin

db.createUser({
  user: "mongoBackup",
  pwd: passwordPrompt(),
  roles: [
    { role: "backup", db: "admin" }
  ]
})
```

Kiểm tra replication lag
```
rs.printSecondaryReplicationInfo()
```

Test ghi và đồng bộ

Trên node 1:
```
use appdb

db.replication_test.insertOne({
  message: "MongoDB replication test",
  createdAt: new Date()
})
```

Trên node 2:
```
mongosh \
  --host mongo02 \
  -u mongoAdmin \
  --authenticationDatabase admin \
  -p
```

```
rs.secondaryOk()
use appdb
db.replication_test.find()
```

test đồng bộ
Node 1:
```
mongosh -u admin-user -p --authenticationDatabase admin
use testdb
db.employees.insert({Name: "Andrew", Age: 25, City: "London", Status: "Married"})
db.employees.find()
```



6. Lưu ý vận hành thực tế cho mô hình 2 node

* Thiếu quorum khi mất 1 node: Replica Set cần đa số voting nodes (N/2). Với 2 node (N=2), đa số là 2. Nếu mongo1 (Primary) sập, mongo2 (Secondary) sẽ giữ ở trạng thái SECONDARY và không tự promote lên Primary để tránh split-brain. Khi mongo1 lên lại, cụm hoạt động bình thường.
* Tạo user riêng cho ứng dụng: Không dùng chung dbadmin cho app, tạo user có quyền readWrite trên đúng database ứng dụng đó.
* Kiểm tra log nếu lỗ khởi động xem: /var/log/mongodb/mongod.log (thường gặp quyền file keyFile khác 400 hoặc sai owner mongodb:mongodb)
* Kiểm tra cú pháp YAML của
```
python3 -c 'import yaml; yaml.safe_load(open("/etc/mongod.conf"))'
sudo grep -nE '^[a-zA-Z]|authorization:|keyFile:|replSetName:|bindIp:' /etc/mongod.conf
```

7. Quy trình Failover khi Node 1 chết

Đây là quy trình vận hành quan trọng nhất của hệ thống 2 node

không chạy force reconfig chỉ vì ứng dụng không kết nối được MongoDB

Bước 1: Xác nhận Node 1 thực sự lỗi
Trên node 2:
```
systemctl status mongod

mongosh \
  --host mongo02 \
  -u mongoAdmin \
  --authenticationDatabase admin \
  -p

rs.status()
```

Node 2 có thể ở trạng thái SECONDARY và không có Primary.

Kiểm tra thời điểm dữ liệu cuối cùng:
```
rs.printSecondaryReplicationInfo()
```

Bước 2: Fence node 1

Trước khi chuyển Node 2 lên Primary, phải đảm bảo Node 1 không thể tiếp tục nhận ghi.

Ưu tiên:
- Tắt VM hoặc máy chủ Node 1.
- Hoặc cô lập hoàn toàn network của Node 1.
- Xác nhận Node 1 không thể phục vụ ứng dụng.

Đây là điều kiện bắt buộc để hạn chế split-brain

Bước 3: Force reconfigure trên node 2

Đăng nhập Node 2 bằng tài khoản quản trị:

```
cfg = {
  _id: "rs0",
  members: [
    {
      _id: 1,
      host: "mongo02:27017",
      votes: 1,
      priority: 1
    }
  ]
}

rs.reconfig(cfg, { force: true })
```
Kiểm tra:
```
db.hello()
```
Mong đợi:
```
{
  isWritablePrimary: true
}
```

Bước 4: Kiểm tra ứng dụng
```
db.adminCommand({ ping: 1 })
```

Bước 5: Khôi phục Node 1

8. Quy trình failback - đưa node 1 trở lại

Giả sử node 2 đang là Primary sau sự cố

Khuyến nghị resync Node 1 sạch từ Node 2 thay vì cố gắng sử dụng lại dữ liệu cũ khi chưa xác định được mức độ lệch dữ liệu.

Bước 1. Chuẩn bị Node 1
```
systemctl stop mongod
```

Giữ lại dữ liệu cũ để đối chiếu:
```
mv /var/lib/mongodb \
   /var/lib/mongodb.pre-resync.$(date +%Y%m%d_%H%M%S)

mkdir -p /var/lib/mongodb

chown mongodb:mongodb /var/lib/mongodb
chmod 700 /var/lib/mongodb
```

Đảm bảo Node 1 vẫn có:
- MongoDB cùng phiên bản với Node 2.
- Cùng Replica Set name rs0.
- Cùng KeyFile.
- Hostname mongo01 phân giải được.

Khởi động lại:
```
systemctl start mongod
```
Bước 2. Add Node 1 từ Node 2

Trên node 2:
```
mongosh \
  --host mongo02 \
  -u mongoAdmin \
  --authenticationDatabase admin \
  -p
```

Thực hiện:
```
rs.add({
  host: "mongo01:27017",
  priority: 0
})
```

Node 1 sẽ bắt đầu initial sync từ node 2

Kiểm tra:
```
rs.status()
```

Chờ đến khi Node 1 chuyển thành SECONDARY.

Bước 3. Đưa Node 1 về cấu hình ưu tiên thấp hơn Node 2

Sau khi đồng bộ hoàn tất:

```
cfg = rs.conf()

cfg.members.forEach(member => {
  if (member.host === "mongo02:27017") {
    member.priority = 2
  }

  if (member.host === "mongo01:27017") {
    member.priority = 1
  }
})

cfg.version++

rs.reconfig(cfg)
```

Node 2 vẫn được ưu tiên làm Primary.
```
rs.status()
```

Hoàn thành Replica Set đã quay lại mô hình 2 node ban đầu.

Chỉ xóa thư mục dữ liệu cũ của Node 1 sau khi đã xác nhận replication ổn định và backup đầy đủ.




