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

После выполнения этой команды иногда все хосты в rs.status() являются SECONDARY. В моем случае помогло rs.stepDown(). Через 10-15 секунд rs.status() показывает, что праймари появился.


Инициируем роутер

```shell
docker exec -it mongos_router mongosh --port 27020

sh.addShard( "shard1/shard1-1:27018,shard1-2:27021,shard1-3:27023");
sh.addShard( "shard2/shard2-1:27019,shard2-2:27022,shard2-3:27024");
exit();
```

После внесения шард выполним sh.status() и увидим примерно следующее
```shell
[direct: mongos] test> sh.status()
shardingVersion
{ _id: 1, clusterId: ObjectId('67d68b8e8819a881ce10baad') }
---
shards
[
  {
    _id: 'shard1',
    host: 'shard1/shard1-1:27018,shard1-2:27021,shard1-3:27023',
    state: 1,
    topologyTime: Timestamp({ t: 1742114273, i: 10 }),
    replSetConfigVersion: Long('-1')
  },
  {
    _id: 'shard2',
    host: 'shard2/shard2-1:27019,shard2-2:27022,shard2-3:27024',
    state: 1,
    topologyTime: Timestamp({ t: 1742114282, i: 9 }),
    replSetConfigVersion: Long('-1')
  }
]
---
active mongoses
[ { '8.0.5': 1 } ]
---
autosplit
{ 'Currently enabled': 'yes' }
---
balancer
{
  'Currently running': 'no',
  'Currently enabled': 'yes',
  'Failed balancer rounds in last 5 attempts': 0,
  'Migration Results for the last 24 hours': 'No recent migrations'
}
---
shardedDataDistribution
[]
---
databases
[
  {
    database: { _id: 'config', primary: 'config', partitioned: true },
    collections: {}
  }
]
[direct: mongos] test>
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

Проверить скорость выполнения можно постманом, первый запрос выполнился за 1.151 сек, второй за 0.040
