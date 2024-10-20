1. Configure the MongoDB repository
First, you need to create a MongoDB repo file to install MongoDB directly from the official MongoDB repository.
```
sudo tee /etc/yum.repos.d/mongodb-org-6.0.repo <<EOF
[mongodb-org-6.0]
name=MongoDB Repository
baseurl=https://repo.mongodb.org/yum/redhat/\$releasever/mongodb-org/6.0/x86_64/
gpgcheck=1
enabled=1
gpgkey=https://www.mongodb.org/static/pgp/server-6.0.asc
EOF
```

2. Install MongoDB 6.0.3
Once the repository is set up, install the mongodb-org package using the dnf command:
```
sudo dnf install -y mongodb-org-6.0.3
```

This will install the MongoDB packages, including:
- mongodb-org-server
- mongodb-org-mongos
- mongodb-org-shell
- mongodb-org-tools

3. Start and Enable MongoDB Service
After installing MongoDB, you need to start the MongoDB service and enable it to start on boot.
```
sudo systemctl start mongod
sudo systemctl enable mongod
```

You can verify that the service is running with:
```
systemctl status mongod
```

4. Verify MongoDB Installation
To check that MongoDB is working properly, connect to the MongoDB shell:
```
mongo --eval 'db.runCommand({ connectionStatus: 1 })'
```

5. Optional: Configure MongoDB (if needed)
By default, MongoDB listens on localhost (127.0.0.1). If you want MongoDB to be accessible from other IP addresses, modify the MongoDB configuration file:

```
nano /etc/mongod.conf
net:
  bindIp: 127.0.0.1,0.0.0.0
```

After modifying the configuration, restart MongoDB:
```
sudo systemctl restart mongod
```

That’s it! You now have MongoDB 6.0.3 installed on CentOS 8.


To create an administrative user for MongoDB with admin privileges, follow these steps:

1. Start the MongoDB Shell
```
mongosh
```
2. Switch to the admin Database
```
use admin
```

3. Create the Admin User
```
db.createUser({
  user: "admin",
  pwd: "admin",
  roles: [{ role: "root", db: "admin" }]
})
```

This command creates a user named admin with the role root, which has full access to all databases and administrative functions.

4. Enable Authentication in MongoDB Configuration
By default, MongoDB does not require authentication. To enable authentication, you need to modify the MongoDB configuration file.

Open the MongoDB configuration file (/etc/mongod.conf) with a text editor like nano or vi:
```
nano /etc/mongod.conf
security:
  authorization: "enabled"
```

Restart MongoDB Service
```
systemctl restart mongod
```

Test the Admin User
```
mongo -u admin -p --authenticationDatabase admin
```

Verify Admin Access
```
db.runCommand({ connectionStatus: 1 })
```

