# mongo-sharding

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


Инициализируем шарды, первый

```shell
docker exec -it shard1 mongosh --port 27018
```

```shell
rs.initiate(
    {
      _id : "shard1",
      members: [
        { _id : 0, host : "shard1:27018" }
      ]
    }
);
```
Не забываем выходить
```shell
exit()
```

Теперь второй

```shell
docker exec -it shard2 mongosh --port 27019
```

```shell
 rs.initiate(
    {
      _id : "shard2",
      members: [
        { _id : 1, host : "shard2:27019" }
      ]
    }
  );
```
Снова выходим
```shell
exit()
```


Инициируем роутер

```shell
docker exec -it mongos_router mongosh --port 27020
```

```shell
sh.addShard( "shard1/shard1:27018");
sh.addShard( "shard2/shard2:27019");
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
docker compose exec -T shard1 mongosh --port 27018
```

```shell
use somedb
db.helloDoc.countDocuments()
```

Результат 492 документа
```shell
shard1 [direct: primary] test> switched to db somedb
shard1 [direct: primary] somedb> 492
```

```shell
docker compose exec -T shard2 mongosh --port 27019
```

```shell
use somedb
db.helloDoc.countDocuments()
```

Результат 508 документов
```shell
shard2 [direct: primary] test> switched to db somedb
shard2 [direct: primary] somedb> 508
```