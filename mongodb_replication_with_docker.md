# MongoDB Replication với Docker

Bước 1: pull images 
``
docker pull mongo
```

Bước 2: Tạo network
```
docker create network my-mongo-cluster
```

Bước 3: Tạo mongo container
```
docker run -p 30001:27017 --name mongo1 --net my-mongo-cluster  mongo mongod --replSet my-mongo-set
docker run -p 30002:27017 --name mongo2 --net my-mongo-cluster  mongo mongod --replSet my-mongo-set
docker run -p 30003:27017 --name mongo3 --net my-mongo-cluster  mongo mongod --replSet my-mongo-set
```

Bước 4: Tạo Replication
```
docker exec -it mongo1 mongo
```

Tạo một bản cài đặt

```
> config = {
        "_id":"my-mongo-set",
        "members":[
            {
                "_id" : 0,
                "host" : "mongo1:27017"
            },
            {
                "_id" : 1,
                "host" : "mongo2:27017"
            },
            {
                "_id" : "3",
                "host" : "mongo3:27017"
            }
        ]
}
```

Thực hiện
```
> rs.initiate(config)
```

Output
```
{ "ok" : 1 }
```
