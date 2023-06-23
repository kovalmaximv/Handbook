# Практика
Запустим Zookeeper:
```bash
./bin/zookeeper-server-start.sh config/zookeeper.properties
```

Запустим Kafka Server:
```bash
./bin/kafka-server-start.sh config/server.properties
```

Мы должны увидеть сообщения, которые скажут о том, что брокер
1. подключился к ZooKeeper: `INFO [ZooKeeperClient Kafka server] Connected. (kafka.zookeeper.ZooKeeperClient)`
2. стартовал и получил идентификатор: `INFO [KafkaServer id=0] started (kafka.server.KafkaServer)`

Создадим топик:
```bash
./bin/kafka-topics.sh --create --topic test --bootstrap-server localhost:9092
```

Посмотрим описание созданного топика:
```bash
./bin/kafka-topics.sh --describe --topic test --bootstrap-server localhost:9092
```

Увидим следующее:
```bash
Topic: test	TopicId: BxJo_cf9QZqPI-CW-scVBQ PartitionCount: 1 	ReplicationFactor: 1	Configs: 
Topic: test	Partition: 0	Leader: 0	Replicas: 0	Isr: 0
```

Запишем сообщение в топик:
```bash
./bin/kafka-console-producer.sh --topic test --bootstrap-server localhost:9092
>Hello world!
```

Прочитаем данные из топика (обратим внимание на конфиг auto.offset.reset):
```bash
./bin/kafka-console-consumer.sh --topic test --bootstrap-server localhost:9092 --consumer-property auto.offset.reset=earliest
```

Удалим топик:
```bash
./bin/kafka-topics.sh --delete --topic test --bootstrap-server localhost:9092
```
