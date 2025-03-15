# mongo-sharding-repl

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Подключаемся к конфиг серверу

```bash
docker exec -it configSrv mongosh --port 27017
```

Инициализируем

```shell
rs.initiate(
     {
      _id : "config_server",
        configsvr: true,
            members: [
              { _id : 0, host : "configSrv:27017" }
            ]
        }
      );
```

Выходим из монгошелла

```shell
exit();
```


Инициализируем шарды + реплики

```shell
docker exec -it shard1-1 mongosh --port 27018

rs.initiate(
    {
      _id : "shard1",
      members: [
        {_id: 0, host: "shard1-1:27018"},
        {_id: 1, host: "shard1-2:27021"},
        {_id: 2, host: "shard1-3:27023"}
      ]
    }
);

exit();
```

```shell
docker exec -it shard2-1 mongosh --port 27019

rs.initiate(
    {
      _id : "shard2",
      members: [
        {_id: 0, host: "shard2-1:27019"},
        {_id: 1, host: "shard2-2:27022"},
        {_id: 2, host: "shard2-3:27024"}
      ]
    }
  );

exit();
```


Инициируем роутер

```shell
docker exec -it mongos_router mongosh --port 27020

sh.addShard( "shard1/shard1-1:27018,shard1-2:27021,shard1-3:27023");
sh.addShard( "shard2/shard2-1:27019,shard2-2:27022,shard2-3:27024");
exit();
```

Сгенерируем моковые данные

```shell
docker exec -it mongos_router mongosh --port 27020
```
```shell
sh.enableSharding("somedb");
sh.shardCollection("somedb.helloDoc", { "name" : "hashed" } )
use somedb
for(var i = 0; i < 1000; i++) db.helloDoc.insertOne({age:i, name:"ly"+i})
```


Теперь сделаем выборку количества документов:

```shell
docker compose exec -T shard1-1 mongosh --port 27018
```

```shell
use somedb
db.helloDoc.countDocuments()
```

Результат 492 документа

```shell
docker compose exec -T shard2-1 mongosh --port 27019
```

```shell
use somedb
db.helloDoc.countDocuments()
```

Результат 508 документов
