# pymongo-api

## Как запустить

```shell
docker compose up -d
```

---

## Инициализация replica set'ов

### 1. Config Server

```shell
docker exec -it configSvr1 mongosh --port 27017

rs.initiate({
  _id: "config_server",
  configsvr: true,
  members: [
    { _id: 0, host: "configSvr1:27017" },
    { _id: 1, host: "configSvr2:27017" },
    { _id: 2, host: "configSvr3:27017" }
  ]
});

exit();
```

### 2. Shard 1

```shell
docker exec -it shard1_1 mongosh --port 27018

rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1_1:27018" },
    { _id: 1, host: "shard1_2:27018" },
    { _id: 2, host: "shard1_3:27018" }
  ]
});

exit();
```

### 3. Shard 2

```shell
docker exec -it shard2_1 mongosh --port 27019

rs.initiate({
  _id: "shard2",
  members: [
    { _id: 0, host: "shard2_1:27019" },
    { _id: 1, host: "shard2_2:27019" },
    { _id: 2, host: "shard2_3:27019" }
  ]
});

exit();
```

## Настройка шардирования

```shell
docker exec -it mongos_router mongosh --port 27020

sh.addShard("shard1/shard1_1:27018,shard1_2:27018,shard1_3:27018");
sh.addShard("shard2/shard2_1:27019,shard2_2:27019,shard2_3:27019");

sh.enableSharding("somedb");

sh.shardCollection("somedb.helloDoc", { "name": "hashed" });

use somedb;
for (var i = 0; i < 1000; i++) db.helloDoc.insert({ age: i, name: "ly" + i });

db.helloDoc.countDocuments();

exit();
```

## Проверка репликации

### Shard 1 — secondary узлы

```shell
docker exec -it shard1_2 mongosh --port 27018

rs.secondaryOk();
use somedb;
db.helloDoc.countDocuments();
exit();
```

```shell
docker exec -it shard1_3 mongosh --port 27018

rs.secondaryOk();
use somedb;
db.helloDoc.countDocuments();
exit();
```

### Shard 2 — secondary узлы

```shell
docker exec -it shard2_2 mongosh --port 27019

rs.secondaryOk();
use somedb;
db.helloDoc.countDocuments();
exit();
```

```shell
docker exec -it shard2_3 mongosh --port 27019

rs.secondaryOk();
use somedb;
db.helloDoc.countDocuments();
exit();
```

## Проверка шардирования

```shell
docker exec -it shard1_1 mongosh --port 27018

use somedb;
db.helloDoc.countDocuments();
exit();
```

```shell
docker exec -it shard2_1 mongosh --port 27019

use somedb;
db.helloDoc.countDocuments();
exit();
```

## Статус

```shell
docker exec -it shard1_1 mongosh --port 27018 --eval "rs.status()"
```

```shell
docker exec -it mongos_router mongosh --port 27020 --eval "sh.status()"
```
