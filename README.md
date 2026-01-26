# pymongo-api

## Как запустить

Запускаем mongodb и приложение

```shell
docker compose up -d
```

Заполняем mongodb данными

```shell
./scripts/mongo-init.sh
```

## Как проверить

### Если вы запускаете проект на локальной машине

Откройте в браузере http://localhost:8080

### Если вы запускаете проект на предоставленной виртуальной машине

Узнать белый ip виртуальной машины

```shell
curl --silent http://ifconfig.me
```

Откройте в браузере http://<ip виртуальной машины>:8080

## Доступные эндпоинты

Список доступных эндпоинтов, swagger http://<ip виртуальной машины>:8080/docs

# Задание 1
### Схемы 
- /diagrams/diagram_1_1.png
- /diagrams/diagram_1_2.png
- /diagrams/diagram_1_3.png

# Задание 2
### Запустить контейнеры
```shell
docker compose -f mongo-sharding.yaml up -d
```

### Настройка сервера конфигурации
```shell
docker compose -f mongo-sharding.yaml exec -T configSrv mongosh --port 27017 --quiet --eval "rs.initiate({_id : 'config_server', configsvr: true, members: [{ _id : 0, host : 'configSrv:27017' }]})"
```

### Настройка шардов
```shell
docker compose -f mongo-sharding.yaml exec -T shard1 mongosh --port 27018 --quiet --eval "rs.initiate({_id : 'shard1', members: [{ _id : 0, host : 'shard1:27018' }]})"
```

```shell
docker compose -f mongo-sharding.yaml exec -T shard2 mongosh --port 27019 --quiet --eval "rs.initiate({_id : 'shard2', members: [{ _id : 0, host : 'shard2:27019' }]})"
```

### Настройка шардов на роутере
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.addShard('shard1/shard1:27018')"
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.addShard('shard2/shard2:27019')"
```

### Настройка шардирования БД
```shell
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.enableSharding('somedb'); sh.shardCollection('somedb.helloDoc', { 'name' : 'hashed' })"
```
### Наполнение БД тестовыми данными
```shell
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "
db.helloDoc.insertOne({name: 'alex', age: 25});
db.helloDoc.insertOne({name: 'maria', age: 30});
db.helloDoc.insertOne({name: 'john', age: 28});
print('Вставлено 3 документа');
"
```
млм или Вставка большего количества (по одному документу)
```shell
for ($i=1; $i -le 10; $i++) {
    docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "db.helloDoc.insertOne({name: 'user_$i', age: $(20+$i), department: 'IT'})"
    Write-Host "Документ $i добавлен"
}
```

### Проверка наполнения БД тестовыми данными
```shell
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "print('Документов в helloDoc: ' + db.helloDoc.countDocuments())"
```

### Проверка состояния БД
```shell
# Количество документов
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "print('Документов в helloDoc: ' + db.helloDoc.countDocuments())"

# Примеры документов
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "db.helloDoc.find().forEach(function(doc) { print(' - ' + doc.name + ' (' + doc.age + ' лет)') })"
```

###  Проверка приложения
http://localhost:8080/helloDoc/users

### Остановка контейнера
```shell
docker compose -f mongo-sharding.yaml down --rmi all --volumes --remove-orphans
```
## Задание 3
### Запуск контейнеров
```bash
docker compose -f mongo-sharding-repl.yaml up -d
```

### Настройка Config Server Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T config-srv1 mongosh --port 27017 --quiet --eval "rs.initiate({ _id: 'config_server', configsvr: true, members: [ { _id: 0, host: 'config-srv1:27017' }, { _id: 1, host: 'config-srv2:27017' }, { _id: 2, host: 'config-srv3:27017' } ] })"
```

### Настройка Shard 1 Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T shard1-node1 mongosh --port 27018 --quiet --eval "rs.initiate({ _id: 'shard1', members: [ { _id: 0, host: 'shard1-node1:27018' }, { _id: 1, host: 'shard1-node2:27018' }, { _id: 2, host: 'shard1-node3:27018' } ] })"
```

### Настройка Shard 2 Replica Set
```shell
docker compose -f mongo-sharding-repl.yaml exec -T shard2-node1 mongosh --port 27019 --quiet --eval "rs.initiate({ _id: 'shard2', members: [ { _id: 0, host: 'shard2-node1:27019' }, { _id: 1, host: 'shard2-node2:27019' }, { _id: 2, host: 'shard2-node3:27019' } ] })"
```

### Добавление шардов в кластер через Mongos
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.addShard('shard1/shard1-node1:27018,shard1-node2:27018,shard1-node3:27018')"
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.addShard('shard2/shard2-node1:27019,shard2-node2:27019,shard2-node3:27019')"
```

### Настройка шардирования для базы данных и наполнение тестовыми данными
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.enableSharding('somedb'); use somedb; db.helloDoc.createIndex({ name: 'hashed' }); sh.shardCollection('somedb.helloDoc', { 'name' : 'hashed' })"
```
### Настройка шардирования для базы данных и наполнение тестовыми данными
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.enableSharding('somedb'); use somedb; db.helloDoc.createIndex({ name: 'hashed' }); sh.shardCollection('somedb.helloDoc', { 'name' : 'hashed' })"
```

### Наполнение тестовыми данными (1000 документов)
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "use somedb; for(var i = 0; i < 100; i++) { db.helloDoc.insertOne({age:i, name:'ly'+i}) }; print('Вставлено 100 документов')"
```
Но в PowerShell для теста я вставку выполняла по одному документу, в цикле не отрабатывает нормально, вставляет, но не сохраняет
```shell
for ($i=1; $i -le 10; $i++) {
    docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "db.helloDoc.insertOne({name: 'user_$i', age: $(20+$i), department: 'IT'})"
    Write-Host "Документ $i добавлен"
}
```

### Проверка состояния БД
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "use somedb; print('Всего документов: ' + db.helloDoc.countDocuments())"
```
### Распределение
```shell
docker compose -f mongo-sharding-repl.yaml exec -T router mongosh --port 27020 --quiet --eval "use somedb; print('=== ПРОВЕРКА ==='); print('Документов: ' + db.helloDoc.countDocuments()); print('Распределение:'); try { printjson(db.helloDoc.getShardDistribution()) } catch(e) { print('Ошибка получения распределения') }"
```

### Проверка приложения
http://localhost:8080/helloDoc/users

### Остановка контейнера
```shell
docker compose -f mongo-sharding-repl.yaml down --rmi all --volumes --remove-orphans
```

---

## Задание 4
### Запуск контейнеров
```shell
docker compose -f mongo-sharding.yaml up -d
```

###  Настройка конфигурационного сервера
```shell
docker compose -f mongo-sharding.yaml exec -T configSrv mongosh --port 27017 --quiet --eval "rs.initiate({_id : 'config_server', configsvr: true, members: [{ _id : 0, host : 'configSrv:27017' }]})"
```
### Настройка шардов
```shell
# Шард 1
docker compose -f mongo-sharding.yaml exec -T shard1 mongosh --port 27018 --quiet --eval "rs.initiate({_id : 'shard1', members: [{ _id : 0, host : 'shard1:27018' }]})"

# Шард 2  
docker compose -f mongo-sharding.yaml exec -T shard2 mongosh --port 27019 --quiet --eval "rs.initiate({_id : 'shard2', members: [{ _id : 0, host : 'shard2:27019' }]})"
```

### Добавление шардов и настройка шардинга
```shell
# Добавляем шарды в роутер
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.addShard('shard1/shard1:27018'); sh.addShard('shard2/shard2:27019')"

# Включаем шардинг для базы данных
docker compose -f mongo-sharding.yaml exec -T router mongosh --port 27020 --quiet --eval "sh.enableSharding('somedb'); sh.shardCollection('somedb.helloDoc', { 'name' : 'hashed' })"
```

### Проверка настройки
```shell
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 --quiet --eval "sh.status()"
```

### Вставка тестовых данных
```shell
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "
db.helloDoc.insertOne({name: 'alex', age: 25});
db.helloDoc.insertOne({name: 'maria', age: 30});
db.helloDoc.insertOne({name: 'john', age: 28});
print('Вставлено 3 документа');
"
```
или Вставка большего количества (по одному документу)

```shell
for ($i=1; $i -le 10; $i++) {
    docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "db.helloDoc.insertOne({name: 'user_$i', age: $(20+$i), department: 'IT'})"
    Write-Host "Документ $i добавлен"
}
```

### Тестирование системы
```shell
# Проверка приложения
curl http://localhost:8080/helloDoc/users -UseBasicParsing

# Или через браузер:
# http://localhost:8080/helloDoc/users
```


### Проверка состояния БД
```shell
# Количество документов
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "print('Документов в helloDoc: ' + db.helloDoc.countDocuments())"

# Примеры документов
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 somedb --eval "db.helloDoc.find().forEach(function(doc) { print(' - ' + doc.name + ' (' + doc.age + ' лет)') })"
```
### Проверка шардинга
```shell
# Статус шардинга
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 --quiet --eval "sh.status()"

# Распределение по шардам
docker compose -f mongo-sharding.yaml exec router mongosh --port 27020 --eval "use somedb; var stats=db.helloDoc.stats(); if(stats.sharded){for(var shard in stats.shards){print(shard+': '+stats.shards[shard].count+' док.')}}"
```

### Проверка API
```shell
# Простая проверка
curl http://localhost:8080/helloDoc/users -UseBasicParsing

# С лимитом
curl "http://localhost:8080/helloDoc/users?limit=3" -UseBasicParsing
```

### Проверку приложения
http://localhost:8080/helloDoc/users

### Остановка контейнеров
```shell
docker compose -f mongo-repl-cache.yaml down --rmi all --volumes --remove-orphans
```

---

## Задание 5
Cхема /diagrams/api_gateway.drawio.png

---

## Задание 6
Cхема /diagrams/CDN.drawio.png
