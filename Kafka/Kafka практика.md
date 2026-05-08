# 📄 Файл: `Message Brokers практика.md`

tags: [message-brokers, devops, kafka, rabbitmq, sqs, queues, pubsub, event-driven, streaming, hands-on]
aliases: [mb-practice, messaging-practice, kafka-practice, rabbitmq-practice]
created: 2026-05-08
---

# 🛠️ Message Brokers для DevOps: Практика

> [!INFO] Структура
> Практические задания разделены по уровням: 🟢 Junior → 🟡 Middle → 🔴 Senior.  
> Каждое задание содержит: задачу, решение, объяснение и DevOps-контекст.

📋 [[#🗂️ Оглавление для навигации|Оглавление]] | [[#🧪 Чек-лист выполнения|Чек-лист]] | [[#🔗 Связь с другими файлами|Связи]]

---

## 🗂️ Оглавление для навигации

### 🟢 Junior (установка, базовые операции, CLI)
- [[#1. Установка RabbitMQ в Docker + базовая настройка|1. RabbitMQ Docker]]
- [[#2. Установка Apache Kafka (KRaft mode) + CLI|2. Kafka KRaft setup]]
- [[#3. Публикация и потребление: RabbitMQ exchanges & queues|3. RabbitMQ pub/sub]]
- [[#4. Kafka Topics, Producers и Consumers через CLI|4. Kafka CLI basics]]
- [[#5. AWS SQS: создание очереди, отправка/получение|5. SQS CLI operations]]
- [[#6. Подключение приложения к брокеру: connection strings, auth|6. App integration]]
- [[#7. Базовый мониторинг: Management UI, JMX, метрики|7. Basic monitoring]]
- [[#8. Dead Letter Queue (DLQ): настройка и отладка|8. DLQ basics]]

### 🟡 Middle (кластеры, персистентность, производительность, security)
- [[#9. Кластер RabbitMQ + Quorum Queues для HA|9. RabbitMQ clustering]]
- [[#10. Kafka Brokers: репликация, ISR, leader election|10. Kafka clustering]]
- [[#11. Consumer Groups: управление offset'ами, rebalancing|11. Consumer groups]]
- [[#12. Настройка retention и персистентности сообщений|12. Retention & persistence]]
- [[#13. Prometheus + Grafana: мониторинг брокеров|13. Advanced monitoring]]
- [[#14. Security: TLS, SASL/SCRAM, ACLs|14. Broker security]]
- [[#15. AWS SQS Advanced: FIFO, visibility timeout, DLQ routing|15. SQS advanced]]
- [[#16. Performance tuning: prefetch, batching, ack modes|16. Performance tuning]]

### 🔴 Senior (масштабирование, production, disaster recovery, cloud)
- [[#17. Kafka Connect + Schema Registry (Avro/Protobuf)|17. Kafka Connect]]
- [[#18. Cross-Cluster Replication: MirrorMaker 2 / RabbitMQ Shovel|18. Cross-cluster sync]]
- [[#19. Partitioning strategy & rebalancing optimization|19. Partitioning]]
- [[#20. Disaster Recovery: backup, multi-AZ, RPO/RTO|20. DR for brokers]]
- [[#21. Auto-scaling consumers: KEDA, dynamic scaling|21. Consumer autoscaling]]
- [[#22. Exactly-Once Semantics: реализация и trade-offs|22. EOS implementation]]
- [[#23. Chaos Engineering: broker failure, network partition tests|23. Chaos testing]]
- [[#24. Tiered Storage и cost optimization для Kafka|24. Kafka tiered storage]]
- [[#25. Multi-protocol gateways: MQTT, AMQP 1.0, STOMP|25. Protocol bridges]]
- [[#26. Cloud-native messaging: SNS, EventBridge, EventGrid patterns|26. Cloud messaging]]

---

## 🟢 Junior (установка, базовые операции, CLI)

### 1. Установка RabbitMQ в Docker + базовая настройка

**Задача**: Развернуть RabbitMQ с management plugin, настроить пользователя и виртуальный хост.

**Решение**:
```bash
# Запуск RabbitMQ с management UI
docker run -d --name rabbitmq \
  -p 5672:5672 -p 15672:15672 \
  -e RABBITMQ_DEFAULT_USER=admin \
  -e RABBITMQ_DEFAULT_PASS=secure_password \
  -v rabbitmq_data:/var/lib/rabbitmq \
  rabbitmq:3.13-management

# Проверка логов и статуса
docker logs rabbitmq
docker exec rabbitmq rabbitmqctl status

# Подключение к CLI внутри контейнера
docker exec -it rabbitmq bash

# Создание vhost и пользователя
rabbitmqctl add_vhost /myapp
rabbitmqctl add_user app_user app_password
rabbitmqctl set_permissions -p /myapp app_user ".*" ".*" ".*"
rabbitmqctl set_user_tags app_user administrator

# Проверка через rabbitmqadmin
rabbitmqadmin -u admin -p secure_password -V /myapp list queues
```

**DevOps-контекст**:
- Management plugin (`:3.13-management`) даёт веб-UI и HTTP API для мониторинга
- `rabbitmqctl set_permissions` использует regex: `".*"` = полный доступ к ресурсам vhost
- Всегда изолируй приложения через separate vhosts
- В production используй `rabbitmq.conf` вместо env vars для сложных настроек

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 2. Установка Apache Kafka (KRaft mode) + CLI

**Задача**: Развернуть Kafka без ZooKeeper (KRaft mode) и выполнить базовые операции через CLI.

**Решение**:
```bash
# Запуск Kafka KRaft в Docker
docker run -d --name kafka \
  -p 9092:9092 -p 9093:9093 \
  -e KAFKA_NODE_ID=1 \
  -e KAFKA_PROCESS_ROLES=broker,controller \
  -e KAFKA_CONTROLLER_QUORUM_VOTERS=1@kafka:9093 \
  -e KAFKA_LISTENERS=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093 \
  -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
  -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
  apache/kafka:3.8.0

# Создание темы
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 \
  --create --topic orders \
  --partitions 3 --replication-factor 1

# Отправка сообщений (Producer)
docker exec -i kafka kafka-console-producer.sh \
  --bootstrap-server localhost:9092 --topic orders
{"order_id": 1, "amount": 100}
{"order_id": 2, "amount": 250}
# Ctrl+D для выхода

# Чтение сообщений (Consumer)
docker exec kafka kafka-console-consumer.sh \
  --bootstrap-server localhost:9092 \
  --topic orders --from-beginning --max-messages 2

# Просмотр информации о теме
docker exec kafka kafka-topics.sh \
  --bootstrap-server localhost:9092 --describe --topic orders
```

**DevOps-контекст**:
- KRaft mode убирает ZooKeeper, упрощая эксплуатацию и снижая footprint
- `--replication-factor=1` только для dev. В production минимум 3
- `--partitions` определяет параллелизм потребителей. Нельзя уменьшить позже
- CLI инструменты (`kafka-topics.sh`, `kafka-console-consumer.sh`) незаменимы для отладки

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 3. Публикация и потребление: RabbitMQ exchanges & queues

**Задача**: Настроить обмен сообщений через `direct` и `topic` exchanges.

**Решение**:
```bash
# Создание exchanges и очередей
rabbitmqadmin declare exchange name=app.direct type=direct durable=true
rabbitmqadmin declare exchange name=app.topic type=topic durable=true

rabbitmqadmin declare queue name=orders.created durable=true
rabbitmqadmin declare queue name=notifications.email durable=true

# Привязка (binding) очередей к exchange с routing key
rabbitmqadmin declare binding source=app.direct destination=orders.created routing_key=order.created
rabbitmqadmin declare binding source=app.topic destination=notifications.email routing_key="notification.*"

# Публикация
rabbitmqadmin publish exchange=app.direct routing_key=order.created payload='{"id": 123}'
rabbitmqadmin publish exchange=app.topic routing_key=notification.email payload='{"user": "alice", "type": "welcome"}'

# Просмотр очередей и сообщений
rabbitmqadmin list queues name messages message_stats.publish_detail.rate
rabbitmqadmin get queue=orders.count count=1
```

**DevOps-контекст**:
- `direct` exchange: точное совпадение routing key. Идеально для command/events
- `topic` exchange: pattern matching (`*` слово, `#` несколько слов). Для фильтрации
- `durable=true`: очередь переживёт перезапуск брокера
- Никогда не публикуй в очередь напрямую — всегда через exchange для гибкости маршрутизации

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 4. Kafka Topics, Producers и Consumers через CLI

**Задача**: Разобраться с partitioning, key-based routing и consumer groups.

**Решение**:
```bash
# Создание темы с ключами и конфигурацией retention
kafka-topics.sh --bootstrap-server localhost:9092 --create \
  --topic user-events \
  --partitions 4 \
  --config retention.ms=604800000 \
  --config cleanup.policy=compact

# Producer с ключом (гарантирует порядок по key в одной partition)
kafka-console-producer.sh --bootstrap-server localhost:9092 --topic user-events \
  --property "parse.key=true" --property "key.separator=:"
alice:{"action": "login"}
bob:{"action": "purchase"}
alice:{"action": "logout"}

# Consumer group с явным указанием ID
kafka-console-consumer.sh --bootstrap-server localhost:9092 \
  --topic user-events --group analytics-group --from-beginning

# Управление consumer groups
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group analytics-group

# Сброс offset'ов (осторожно!)
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group analytics-group --topic user-events --reset-offsets --to-latest --execute
```

**DevOps-контекст**:
- `key` определяет partition: одинаковые ключи → одинаковая partition → порядок гарантирован
- `cleanup.policy=compact`: хранит только последнее значение по ключу (stateful streaming)
- Consumer group балансирует partition'ы между инстансами. Макс параллелизм = кол-во partition
- `--reset-offsets` опасен в production. Всегда делай snapshot состояния перед сбросом

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 5. AWS SQS: создание очереди, отправка/получение

**Задача**: Работа с SQS через AWS CLI, настройка базовых параметров.

**Решение**:
```bash
# Создание стандартной очереди
aws sqs create-queue --queue-name orders-processing \
  --attributes '{
    "DelaySeconds": "0",
    "MaximumMessageSize": "262144",
    "MessageRetentionPeriod": "1209600",
    "ReceiveMessageWaitTimeSeconds": "20"
  }'

# Получение URL очереди
QUEUE_URL=$(aws sqs get-queue-url --queue-name orders-processing --query 'QueueUrl' --output text)

# Отправка сообщения
aws sqs send-message \
  --queue-url $QUEUE_URL \
  --message-body '{"order_id": 456, "status": "pending"}' \
  --message-group-id "orders"

# Получение сообщений (long polling)
aws sqs receive-message \
  --queue-url $QUEUE_URL \
  --max-number-of-messages 10 \
  --wait-time-seconds 20 \
  --visibility-timeout 30

# Удаление сообщения после обработки (обязательно!)
RECEIPT_HANDLE=$(aws sqs receive-message ... --query 'Messages[0].ReceiptHandle' --output text)
aws sqs delete-message --queue-url $QUEUE_URL --receipt-handle $RECEIPT_HANDLE

# Просмотр метрик (требует CloudWatch)
aws cloudwatch get-metric-statistics \
  --namespace AWS/SQS \
  --metric-name ApproximateNumberOfMessagesVisible \
  --dimensions Name=QueueName,Value=orders-processing \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --end-time $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 300 --statistics Average
```

**DevOps-контекст**:
- `ReceiveMessageWaitTimeSeconds` (long polling) снижает пустые запросы и стоимость
- `visibility-timeout`: время до возврата сообщения в очередь, если не удалено. Подбирай под макс время обработки
- SQS не гарантирует порядок (кроме FIFO). Для ordered processing используй `FIFO` очереди
- Всегда удаляй сообщения после успешной обработки, иначе будет infinite loop

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 6. Подключение приложения к брокеру: connection strings, auth

**Задача**: Настроить безопасное подключение из Python-приложения.

**Решение**:
```python
# pip install pika kafka-python-boto3

# RabbitMQ (pika)
import pika, os

credentials = pika.PlainCredentials(
    os.getenv("RABBITMQ_USER", "app_user"),
    os.getenv("RABBITMQ_PASS", "secret")
)
params = pika.ConnectionParameters(
    host=os.getenv("RABBITMQ_HOST", "localhost"),
    port=5672,
    virtual_host=os.getenv("RABBITMQ_VHOST", "/myapp"),
    credentials=credentials,
    heartbeat=600,
    blocked_connection_timeout=300
)

with pika.BlockingConnection(params) as conn:
    channel = conn.channel()
    channel.basic_publish(
        exchange="app.direct",
        routing_key="order.created",
        body=b'{"order_id": 789}',
        properties=pika.BasicProperties(delivery_mode=2)  # persistent
    )

# Kafka (confluent-kafka)
from confluent_kafka import Producer, Consumer
import json

producer = Producer({
    'bootstrap.servers': os.getenv("KAFKA_BOOTSTRAP", "localhost:9092"),
    'security.protocol': 'SASL_PLAINTEXT',
    'sasl.mechanism': 'SCRAM-SHA-256',
    'sasl.username': os.getenv("KAFKA_USER"),
    'sasl.password': os.getenv("KAFKA_PASS"),
    'acks': 'all',
    'retries': 3
})

producer.produce('orders', key='user_123', value=json.dumps({"action": "buy"}))
producer.flush()

# SQS (boto3)
import boto3, os
from botocore.config import Config

sqs = boto3.client('sqs', 
    region_name=os.getenv("AWS_REGION", "us-east-1"),
    config=Config(retries={'max_attempts': 3, 'mode': 'adaptive'})
)

resp = sqs.receive_message(
    QueueUrl=os.getenv("SQS_URL"),
    MaxNumberOfMessages=10,
    WaitTimeSeconds=10
)
```

**DevOps-контекст**:
- Всегда читай креды из env vars или secrets manager, никогда не хардкодь
- `heartbeat` и `blocked_connection_timeout` предотвращают zombie-соединения
- Kafka `acks: 'all'` + `retries` гарантируют delivery, но снижают throughput
- Boto3 `Config` с adaptive retries критичен для production при throttle SQS

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 7. Базовый мониторинг: Management UI, JMX, метрики

**Задача**: Снять ключевые метрики брокеров для health-check.

**Решение**:
```bash
# RabbitMQ: HTTP API мониторинг
curl -s -u admin:secure_password http://localhost:15672/api/overview | jq '.queue_totals.messages'
curl -s -u admin:secure_password http://localhost:15672/api/health/checks/virtual-hosts

# RabbitMQ: CLI health check
docker exec rabbitmq rabbitmqctl eval 'rabbit_health_check:check()'

# Kafka: JMX метрики (включи JMX в docker: -e KAFKA_JMX_PORT=9999 -p 9999:9999)
# Использование jmx_exporter для Prometheus (рекомендуется)
docker exec kafka jcmd 1 VM.flags | grep jmx

# Kafka: CLI проверка лага
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group analytics-group | awk '$6 != "0" {print $1, $6}'

# SQS: CloudWatch алерты (CLI)
aws cloudwatch put-metric-alarm \
  --alarm-name "SQS-Backlog-High" \
  --metric-name ApproximateNumberOfMessagesVisible \
  --namespace AWS/SQS \
  --threshold 1000 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 2 \
  --period 300 \
  --statistic Average \
  --alarm-actions "arn:aws:sns:us-east-1:123456789012:alert-topic" \
  --dimensions Name=QueueName,Value=orders-processing
```

**DevOps-контекст**:
- RabbitMQ Management API даёт JSON для интеграции с любыми мониторинговыми системами
- Kafka lag = `log-end-offset` - `current-offset`. Рост лага = bottleneck в потребителях
- JMX без `jmx_exporter` сложен в продакшене. Используй sidecar-контейнер с exporter
- SQS метрики доступны только в CloudWatch. Агрегируй их в Prometheus через YACE exporter

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 8. Dead Letter Queue (DLQ): настройка и отладка

**Задача**: Настроить автоматическую маршрутизацию failed сообщений в DLQ.

**Решение**:
```bash
# RabbitMQ DLQ
rabbitmqadmin declare queue name=orders.dlq durable=true
rabbitmqadmin declare exchange name=orders.dlx type=direct durable=true
rabbitmqadmin declare binding source=orders.dlx destination=orders.dlq routing_key=failed

# Основная очередь с DLX policy
rabbitmqadmin declare queue name=orders.main durable=true \
  arguments='{"x-dead-letter-exchange": "orders.dlx", "x-dead-letter-routing-key": "failed", "x-message-ttl": 86400000}'

# Kafka DLQ (через Streams или приложение)
# Создаём отдельную тему для failed messages
kafka-topics.sh --bootstrap-server localhost:9092 --create --topic orders.dlq --partitions 3

# В коде потребителя:
# try: process(msg)
# except: producer.send("orders.dlq", value=msg, headers=[("error", b"timeout")])

# SQS Redrive Policy (DLQ через JSON)
QUEUE_URL=$(aws sqs get-queue-url --queue-name orders-processing --query QueueUrl -o text)
DLQ_URL=$(aws sqs get-queue-url --queue-name orders.dlq --query QueueUrl -o text)

aws sqs set-queue-attributes \
  --queue-url $QUEUE_URL \
  --attributes '{
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123456789012:orders.dlq\",\"maxReceiveCount\":3}"
  }'
```

**DevOps-контекст**:
- DLQ предотвращает poison pill messages (бесконечный retry одного битого сообщения)
- `x-message-ttl` в RabbitMQ ограничивает время жизни сообщения до попадания в DLQ
- В Kafka DLQ реализуется на уровне приложения или Kafka Streams (NoRetry topic pattern)
- SQS `maxReceiveCount` определяет, сколько раз сообщение будет retried до DLQ
- Всегда мониторь размер DLQ. Рост = баг в processing logic

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🟡 Middle (кластеры, персистентность, производительность, security)

### 9. Кластер RabbitMQ + Quorum Queues для HA

**Задача**: Развернуть 3-нодовый кластер RabbitMQ с отказоустойчивыми очередями.

**Решение**:
```yaml
# docker-compose.yml
version: '3.8'
services:
  rabbit1:
    image: rabbitmq:3.13-management
    hostname: rabbit1
    environment:
      RABBITMQ_ERLANG_COOKIE: "devops_secret_cookie"
      RABBITMQ_NODENAME: rabbit@rabbit1
    ports: ["5672:5672", "15672:15672"]
    volumes: [rabbit1_/var/lib/rabbitmq]
  rabbit2:
    image: rabbitmq:3.13-management
    hostname: rabbit2
    environment:
      RABBITMQ_ERLANG_COOKIE: "devops_secret_cookie"
      RABBITMQ_NODENAME: rabbit@rabbit2
    ports: ["5673:5672"]
  rabbit3:
    image: rabbitmq:3.13-management
    hostname: rabbit3
    environment:
      RABBITMQ_ERLANG_COOKIE: "devops_secret_cookie"
      RABBITMQ_NODENAME: rabbit@rabbit3
    ports: ["5674:5672"]

volumes: {rabbit1_}
```

```bash
# Формирование кластера
docker exec rabbit2 rabbitmqctl stop_app
docker exec rabbit2 rabbitmqctl join_cluster rabbit@rabbit1
docker exec rabbit2 rabbitmqctl start_app

docker exec rabbit3 rabbitmqctl stop_app
docker exec rabbit3 rabbitmqctl join_cluster rabbit@rabbit1 --ram
docker exec rabbit3 rabbitmqctl start_app

# Создание Quorum Queue (RAFT-based, modern HA)
rabbitmqctl add_queue orders.quorum quorum --quorum

# Проверка статуса кластера и очередей
rabbitmqctl cluster_status
rabbitmqctl list_queues name type policy durable
```

**DevOps-контекст**:
- `ERLANG_COOKIE` должен быть идентичен на всех нодах. Это ключ кластеризации
- Quorum Queues (Quorum) заменяют classic mirrored queues. Гарантируют durability и safety
- `--ram` ноды хранят метаданные в памяти. Используй для stateless routing, не для critical data
- В production размещай ноды в разных AZ. RabbitMQ синхронно реплицирует quorum queues

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 10. Kafka Brokers: репликация, ISR, leader election

**Задача**: Настроить 3-брокер кластер Kafka с правильной репликацией и monitoring ISR.

**Решение**:
```bash
# Запуск кластера через docker-compose (kafkakraft)
# server.properties (ключевые параметры):
# broker.id=1,2,3
# listeners=PLAINTEXT://:9092
# num.partitions=3
# default.replication.factor=3
# min.insync.replicas=2

# Проверка репликации и ISR
kafka-topics.sh --bootstrap-server kafka1:9092 --describe --topic orders
# Topic: orders  PartitionCount:3  ReplicationFactor:3
#   Partition: 0  Leader: 1  Replicas: 1,2,3  Isr: 1,2,3
#   Partition: 1  Leader: 2  Replicas: 2,3,1  Isr: 2,3,1
#   Partition: 2  Leader: 3  Replicas: 3,1,2  Isr: 3,1,2

# Проверка under-replicated partitions (критично!)
kafka-topics.sh --bootstrap-server localhost:9092 --describe --under-replicated-partitions

# Изменение replication factor на лету
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
  --reassignment-json-file reassign.json --execute

# reassign.json
{"version":1, "partitions": [{"topic":"orders","partition":0,"replicas":[1,2,3]}]}
```

**DevOps-контекст**:
- `min.insync.replicas=2` + `acks=all` = guarantee durability при падении 1 брокера
- `ISR` (In-Sync Replicas): только они участвуют в leader election. Если ISR < min.insync → producers блокируются
- `under-replicated-partitions` > 0 = тревога. Означает, что репликация не успевает
- Балансировка лидеров (`kafka-leader-election.sh`) нужна после добавления/удаления брокеров

[[#🗂️ Оглавлению для навигации|↑ К оглавлению]]

### 11. Consumer Groups: управление offset'ами, rebalancing

**Задача**: Настроить consumer groups, отследить лаг, выполнить graceful shutdown.

**Решение**:
```bash
# Просмотр статуса всех групп
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Детальный статус: lag, offset, consumer ID
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --group payment-processors

# Отключение consumer (кооперативный rebalancing)
# В коде: consumer.unsubscribe() + consumer.close() (отправляет LeaveGroup)

# Принудительный reset offset (осторожно!)
# to-earliest, to-latest, to-offset N, shift-by +N/-N
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --group payment-processors --topic payments --reset-offsets --to-latest --dry-run
# Проверь вывод, потом замени --dry-run на --execute

# Отключение sticky assignment (уменьшает rebalance storms)
# В consumer config: partition.assignment.strategy=org.apache.kafka.clients.consumer.CooperativeStickyAssignor
```

**DevOps-контекст**:
- Lag = `log-end-offset` - `committed-offset`. Растущий lag = потребители не справляются
- `CooperativeStickyAssignor` уменьшает downtime при rebalance (incremental instead of stop-the-world)
- Никогда не делай `--reset-offsets --execute` без dry-run и бэкапа текущего состояния
- Мониторь `group-coordinator` и `rebalance` метрики в Kafka. Частые ребалансы = unstable consumers

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 12. Настройка retention и персистентности сообщений

**Задача**: Сконфигурировать политики хранения для разных типов данных.

**Решение**:
```bash
# Kafka: Time & Size based retention
kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --topic orders --add-config retention.ms=259200000,retention.bytes=1073741824

# Kafka: Compaction (для state/changelog)
kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --topic user-profiles --add-config cleanup.policy=compact,segment.bytes=104857600

# RabbitMQ: TTL и max-length (queue policies)
rabbitmqctl set_policy ttl_policy "^orders\." \
  '{"message-ttl":86400000,"max-length":100000,"overflow":"reject-publish"}' --apply-to queues

# Проверка применения
kafka-configs.sh --bootstrap-server localhost:9092 --describe --topic orders
rabbitmqctl list_policies
```

**DevOps-контекст**:
- `retention.bytes` > 0 предотвращает заполнение дисков. При превышении старые сегменты удаляются
- `cleanup.policy=compact` + `segment.bytes` оптимизирует storage для stateful streams
- RabbitMQ `overflow: reject-publish` возвращает ошибку producer'у при переполнении (backpressure)
- В Kafka retention применяется к сегментам. Реальное удаление происходит при log cleanup, не мгновенно

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 13. Prometheus + Grafana: мониторинг брокеров

**Задача**: Развернуть экспортёры метрик и базовые дашборды.

**Решение**:
```yaml
# docker-compose добавление exporters
services:
  rabbitmq-exporter:
    image: kbudde/rabbitmq-exporter
    environment:
      RABBITMQ_URL: http://rabbit1:15672
      RABBITMQ_USER: admin
      RABBITMQ_PASSWORD: secure_password
    ports: ["9419:9419"]

  kafka-jmx-exporter:
    image: bitnami/jmx-exporter:latest
    environment:
      JMX_PORT: 9999
      KAFKA_BROKER_ID: 1
    ports: ["5556:5556"]
    volumes:
      - ./jmx_config.yaml:/opt/bitnami/jmx-exporter/config.yaml
```

```yaml
# jmx_config.yaml (kafka)
hostPort: localhost:9999
rules:
  - pattern: kafka.server<type=ReplicaManager, name=IsrShrinksPerSec><>Value
    name: kafka_isr_shrinks
  - pattern: kafka.server<type=BrokerTopicMetrics, name=BytesInPerSec><>Count
    name: kafka_bytes_in_total
    type: COUNTER
```

**Grafana Dashboards ID**:
- RabbitMQ: `10991` (RabbitMQ-Overview)
- Kafka: `7589` (Kafka Cluster Monitoring)
- SQS: YACE exporter + `14835` (AWS SQS)

**DevOps-контекст**:
- RabbitMQ exporter использует Management API. Не требует JMX
- Kafka требует `jmx-exporter` для Prometheus. Native Prometheus metrics доступны с Kafka 3.x+
- Ключевые алерты: `node_disk_usage > 85%`, `kafka_under_replicated_partitions > 0`, `rabbitmq_queue_messages_ready > 100000`
- Храни retention метрик в Prometheus минимум 30 дней для capacity planning

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 14. Security: TLS, SASL/SCRAM, ACLs

**Задача**: Настроить шифрование трафика и авторизацию на уровне ресурсов.

**Решение**:
```bash
# RabbitMQ TLS + SASL
# rabbitmq.conf
listeners.ssl.default = 5671
ssl_certfile = /etc/rabbitmq/cert.pem
ssl_keyfile = /etc/rabbitmq/key.pem
ssl_cacertfile = /etc/rabbitmq/ca.pem
authentication_mechanisms.1 = PLAIN
authentication_mechanisms.2 = AMQPLAIN

# Создание SASL/SCRAM пользователя (Kafka)
kafka-configs.sh --bootstrap-server localhost:9092 --alter \
  --add-config 'SCRAM-SHA-256=[password=secure123]' --entity-type users --entity-name app_user

# Kafka ACLs
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:app_user --operation Write --topic orders
kafka-acls.sh --bootstrap-server localhost:9092 --add \
  --allow-principal User:app_user --operation Read --topic orders --group analytics-group
kafka-acls.sh --bootstrap-server localhost:9092 --list

# SQS KMS encryption (at rest)
aws sqs set-queue-attributes --queue-url $QUEUE_URL \
  --attributes '{"KmsMasterKeyId": "alias/sqs-key", "KmsDataKeyReusePeriodSeconds": 300}'
```

**DevOps-контекст**:
- TLS обязателен для трафика между app ↔ broker. Internal cluster traffic тоже шифруй
- SASL/SCRAM безопаснее PLAIN. Используй интеграцию с LDAP/OIDC для enterprise
- ACLs: принцип минимальных привилегий. Producer → Write, Consumer → Read, Admin → Cluster
- SQS KMS encrypts at rest. In transit всегда TLS. `KmsDataKeyReusePeriod` балансирует security/performance

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 15. AWS SQS Advanced: FIFO, visibility timeout, DLQ routing

**Задача**: Настроить FIFO очередь с message deduplication и продвинутым retry.

**Решение**:
```bash
# Создание FIFO очереди
aws sqs create-queue --queue-name orders.fifo.fifo \
  --attributes '{
    "FifoQueue": "true",
    "ContentBasedDeduplication": "true",
    "MessageDeduplicationId": "auto",
    "MessageGroupId": "orders",
    "ReceiveMessageWaitTimeSeconds": "20",
    "VisibilityTimeout": "120"
  }'

# Отправка с явным deduplication ID (если ContentBasedDeduplication=false)
aws sqs send-message \
  --queue-url https://sqs.us-east-1.amazonaws.com/123456789012/orders.fifo.fifo \
  --message-body '{"id": 1}' \
  --message-group-id "order-process" \
  --message-deduplication-id "uniq-$(uuidgen)"

# DLQ с exponential backoff (через Lambda или Step Functions)
# В SQS нет нативного exponential backoff. Паттерн:
# Queue 1 (delay=0) → fail → Queue 2 (delay=30) → fail → Queue 3 (delay=300) → DLQ

aws sqs set-queue-attributes --queue-url $QUEUE2_URL \
  --attributes '{"DelaySeconds": "30"}'
```

**DevOps-контекст**:
- FIFO гарантирует порядок только внутри `MessageGroupId`. Разные группы обрабатываются параллельно
- `ContentBasedDeduplication` считает SHA-256 body. Для уникальности добавляй timestamp/UUID
- SQS visibility timeout должно быть > max processing time. Иначе сообщение будет processed дважды
- Для backoff в SQS используй dead-letter chains с разными `DelaySeconds` или Step Functions

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 16. Performance tuning: prefetch, batching, ack modes

**Задача**: Оптимизировать throughput и latency потребителей.

**Решение**:
```yaml
# RabbitMQ QoS (prefetch)
# channel.basic_qos(prefetch_count=10)
# 10 = consumer получит макс 10 unacked messages. Баланс между latency и throughput

# Kafka Batch & Compression
# producer.properties
batch.size=16384          # 16KB batch
linger.ms=5               # ждём 5ms для заполнения batch
compression.type=lz4      # или zstd (лучший ratio/speed)
buffer.memory=33554432    # 32MB memory buffer

# Kafka Consumer Tuning
# consumer.properties
fetch.min.bytes=1024      # мин байт для fetch
fetch.max.wait.ms=100     # макс ожидание
max.poll.records=500      # батч за один poll
enable.auto.commit=false  # manual commit для exactly-once

# SQS Long Polling + Batching
# boto3 client
sqs.receive_message(
    QueueUrl=url,
    MaxNumberOfMessages=10,
    WaitTimeSeconds=20,
    VisibilityTimeout=60
)
```

**DevOps-контекст**:
- `prefetch_count=1` = fair dispatch, но ниже throughput. `10-50` = оптимум для большинства workload
- Kafka `batch.size` + `linger.ms` = trade-off latency vs throughput. Для real-time ставь `linger.ms=0`
- `compression.type=lz4` снижает network I/O и storage на 30-50% при minimal CPU overhead
- SQS `WaitTimeSeconds=20` (long polling) снижает cost и пустые responses на 90%+

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🔴 Senior (масштабирование, production, disaster recovery, cloud)

### 17. Kafka Connect & Schema Registry (Avro/Protobuf)

**Задача**: Настроить распределённый Kafka Connect с Schema Registry для evolution-safe данных.

**Решение**:
```yaml
# docker-compose: schema-registry + connect
services:
  schema-registry:
    image: confluentinc/cp-schema-registry:7.6.0
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka1:9092,kafka2:9092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081
    ports: ["8081:8081"]

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.6.0
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka1:9092,kafka2:9092
      CONNECT_GROUP_ID: connect-cluster
      CONNECT_CONFIG_STORAGE_TOPIC: connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: connect-status
      CONNECT_KEY_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_KEY_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
    ports: ["8083:8083"]
```

```bash
# Регистрация схемы
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  http://localhost:8081/subjects/orders-value/versions \
  -d '{"schema": "{\"type\":\"record\",\"name\":\"Order\",\"fields\":[{\"name\":\"id\",\"type\":\"int\"},{\"name\":\"amount\",\"type\":\"double\"}]}"}'

# Проверка совместимости
curl http://localhost:8081/config/orders-value
# compatibility.level = BACKWARD (default)

# Запуск connector
curl -X POST http://localhost:8083/connectors \
  -H "Content-Type: application/json" \
  -d '{
    "name": "jdbc-sink",
    "config": {
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
      "tasks.max": "2",
      "topics": "orders",
      "connection.url": "jdbc:postgresql://db:5432/app",
      "auto.create": "true",
      "insert.mode": "upsert",
      "pk.fields": "id"
    }
  }'
```

**DevOps-контекст**:
- Schema Registry гарантирует совместимость схем. `BACKWARD` = новые readers читают старые данные
- Kafka Connect state хранится в internal topics (`connect-configs`, `offsets`, `status`). Backup'и их!
- Avro/Protobuf сокращают payload size на 40-70% vs JSON. Требуют registry для десериализации
- `tasks.max` определяет параллелизм sink/source. Привязывай к количеству partitions topic'а

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 18. Cross-Cluster Replication: MirrorMaker 2 / RabbitMQ Shovel

**Задача**: Настроить репликацию между production и DR кластерами.

**Решение**:
```properties
# Kafka MirrorMaker 2 (connect-distributed.properties)
clusters=source, target
source.bootstrap.servers=prod-kafka:9092
target.bootstrap.servers=dr-kafka:9092

# Replication flows
source->target.enabled=true
source->target.topics=orders.*, payments.*
source->target.groups=.*
source->target.configs.replication.factor=3
source->target.configs.sync.group.offsets.enabled=true

# Запуск
connect-mirror-maker.sh mm2.properties

# RabbitMQ Shovel plugin
rabbitmq-plugins enable rabbitmq_shovel
rabbitmq-plugins enable rabbitmq_shovel_management

# rabbitmq.conf (shovel static)
shovel.static-shovels.1.name = prod-to-dr
shovel.static-shovels.1.source-uri = amqp://prod-user:pass@prod-rabbit:5672
shovel.static-shovels.1.source-queue = orders
shovel.static-shovels.1.dest-uri = amqp://dr-user:pass@dr-rabbit:5672
shovel.static-shovels.1.dest-queue = orders
shovel.static-shovels.1.add-forward-headers = true
```

**DevOps-контекст**:
- MM2 реплицирует не только сообщения, но и consumer offsets (критично для seamless failover)
- Shovel работает на уровне queues. MM2 на уровне topics. Оба support active-active с careful routing
- Мониторь `ReplicationLatency` и `EndToEndLatency`. Рост = network congestion или underpowered DR cluster
- В DR сценарии: сначала переведи consumers, потом producers. Избегай split-brain

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 19. Partitioning strategy & rebalancing optimization

**Задача**: Спроектировать оптимальную partition схему для high-throughput системы.

**Решение**:
```yaml
# Формула: Partitions = (Target Throughput / Consumer Throughput) * Safety Factor
# Пример: 100k msg/sec producer, consumer handles 20k → 5 partitions * 1.5 safety = 8

# Kafka: Custom Partitioner
# class UserPartitioner implements Partitioner {
#   public int partition(String topic, Object key, byte[] keyBytes, ...) {
#     return Math.abs(key.hashCode()) % numPartitions;
#   }
# }

# Балансировка партиций (Rack Awareness)
broker.rack=us-east-1a
# replica.selector.class=org.apache.kafka.common.replica.RackAwareReplicaSelector
# Гарантирует, что реплики в разных AZ

# Rebalancing optimization
group.instance.id=consumer-1-static  # Static membership (skip rebalance on restart)
session.timeout.ms=45000
heartbeat.interval.ms=15000
```

**DevOps-контекст**:
- Уменьшить количество partition'ов невозможно. Только создать новую тему и migrate данные
- `group.instance.id` предотвращает unnecessary rebalances при rolling deployments consumers
- Rack awareness распределяет реплики по AZ. Критично для HA при zone outage
- Partition skew (неравномерное распределение key) = hotspot. Используй composite keys или hashing

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 20. Disaster Recovery: backup, multi-AZ, RPO/RTO

**Задача**: Спроектировать DR план для message brokers с RPO < 5min, RTO < 30min.

**Решение**:
```yaml
# Kafka DR Strategy
# 1. Multi-AZ deployment (broker.rack)
# 2. MirrorMaker 2 async replication to DR region
# 3. Backup critical topics:
#    kafka-dump-log.sh --cluster-log-dir /var/kafka-logs --print-data-log > backup.log
# 4. Schema Registry backup (DB dump + schema files)

# RabbitMQ DR Strategy
# 1. Quorum queues + mirrored policies
# 2. RabbitMQ Shovel to DR cluster
# 3. Definitions backup:
#    rabbitmqctl export_definitions /backup/rabbitmq_definitions.json
#    rabbitmqctl import_definitions /backup/rabbitmq_definitions.json

# SQS DR: AWS native multi-AZ
# SQS автоматически реплицируется в 3 AZ региона
# Cross-region: Lambda trigger → SNS → SQS в DR регионе
# RPO зависит от replication lag, обычно < 1 мин

# Тестирование DR (Chaos)
# 1. Отключить 1 AZ (simulated)
# 2. Проверить failover producers/consumers
# 3. Замерить data loss (RPO) и recovery time (RTO)
```

**DevOps-контекст**:
- RPO (Recovery Point Objective) = допустимая потеря данных. Зависит от sync/async replication
- RTO (Recovery Time Objective) = время восстановления. Зависит от automation и runbooks
- Kafka `acks=all` + `min.insync.replicas=2` даёт RPO=0 при отказе 1 брокера
- Всегда тестируй DR quarterly. Backup без restore test = иллюзия безопасности

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 21. Auto-scaling consumers: KEDA, dynamic scaling

**Задача**: Настроить автомасштабирование потребителей на основе lag очереди.

**Решение**:
```yaml
# KEDA ScaledObject (Kubernetes)
apiVersion: keda.sh/v1alpha1
kind: ScaledObject
meta
  name: kafka-consumer-scaler
spec:
  scaleTargetRef:
    name: order-processor-deployment
  minReplicaCount: 2
  maxReplicaCount: 20
  pollingInterval: 15
  triggers:
  - type: kafka
    meta
      bootstrapServers: kafka-cluster:9092
      consumerGroup: payment-processors
      topic: payments
      lagThreshold: "1000"  # Scale up if lag > 1000
      allowIdleConsumers: "true"
  - type: prometheus
    metadata:
      serverAddress: http://prometheus:9090
      metricName: kafka_consumer_lag
      threshold: "500"

# RabbitMQ Scaling
# KEDA trigger: rabbitmq
# metadata:
#   hostFromEnv: RABBITMQ_HOST
#   queueName: orders
#   queueLength: "100"

# SQS Scaling (AWS Native)
# Application Auto Scaling + CloudWatch Metric: ApproximateNumberOfMessagesVisible
# TargetTrackingScalingPolicyConfiguration:
#   TargetValue: 50.0
#   PredefinedMetricSpecification: PredefinedMetricType: ASGAverageCPUUtilization
```

**DevOps-контекст**:
- KEDA queries broker metrics directly. Не требует изменения кода приложения
- `lagThreshold` подбирай экспериментально. Слишком низкий = thrashing, высокий = backlog
- `maxReplicaCount` ограничивай по consumer throughput и broker connection limits
- SQS auto-scaling работает лучше с Lambda. Для ECS/EKS используй KEDA или custom controller

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 22. Exactly-Once Semantics: реализация и trade-offs

**Задача**: Настроить EOS в Kafka и понять ограничения.

**Решение**:
```yaml
# Producer config (EOSv2 / Transactional)
enable.idempotence=true
acks=all
transactional.id=my-app-tx-1
isolation.level=read_committed  # Consumer side

# Код транзакционного producer (Java/Pseudo)
producer.initTransactions()
try {
  producer.beginTransaction()
  producer.send(topic1, msg1)
  producer.send(topic2, msg2)
  producer.commitTransaction()
} catch (Exception e) {
  producer.abortTransaction()
}

# Streams config (EOS по умолчанию)
processing.guarantee=exactly_once_v2
num.standby.replicas=1  # Для low-latency failover
```

**Trade-offs**:
```
✅ Плюсы:
   • Нет дубликатов при restart/failure
   • Атомарность multi-topic writes
   • Критично для financial systems, inventory

❌ Минусы:
   • Throughput снижается на 20-40% (overhead transaction logs)
   • Latency увеличивается (commit latency)
   • Требует строгого ordering и state management
   • Не поддерживается в RabbitMQ/SQS на уровне брокера (только app-level idempotency)
```

**DevOps-контекст**:
- EOS в Kafka = `enable.idempotence` + `transactional.id`. Включай только когда бизнес-логика требует
- Для RabbitMQ/SQS используй app-level idempotency keys + deduplication tables (Redis/DB)
- `isolation.level=read_committed` скрывает uncommitted messages от consumers
- Мониторь `transaction-abort-rate`. Высокий = instability или misconfiguration

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 23. Chaos Engineering: broker failure, network partition tests

**Задача**: Протестировать resilience brokers в production-like среде.

**Решение**:
```bash
# Kafka: Leader failure test
# 1. Найти лидера partition
kafka-topics.sh --describe --topic orders | grep "Leader: 1"
# 2. Остановить broker 1
docker stop kafka1
# 3. Проверить failover
kafka-topics.sh --describe --topic orders | grep "Leader: 2"
# 4. Проверить lag consumers (должен стабилизироваться)

# RabbitMQ: Network partition simulation
# 1. Отключить сеть между node1 и node2
tc qdisc add dev eth0 root netem delay 10000ms
# 2. Проверить partition handling
rabbitmqctl cluster_status
# 3. Очистить partition (autoheal или pause_minority)
rabbitmqctl start_app  # после восстановления сети

# SQS: Throttle simulation
# AWS не даёт симулировать throttle. Используй localstack + rate limiting proxy
# Или inject delay в consumer: time.sleep(5) + observe visibility timeout behavior

# Чек-лист resilience тестов:
# ✅ Producer continues after broker failure?
# ✅ Consumers rebalance without data loss?
# ✅ DLQ captures poison messages?
# ✅ Lag returns to normal within SLA?
```

**DevOps-контекст**:
- Chaos engineering должен проводиться в staging, но с production data patterns
- RabbitMQ `pause_minority` останавливает minority nodes при partition. Предотвращает split-brain
- Kafka автоматически re-elects leader из ISR. Время failover = `leader.imbalance.check.interval`
- Всегда имей rollback plan. Chaos test = learning, не production incident

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 24. Tiered Storage и cost optimization для Kafka

**Задача**: Настроить tiered storage для снижения cost при long retention.

**Решение**:
```properties
# server.properties (Kafka 3.6+)
remote.log.storage.system.enable=true
remote.log.manager.thread.pool.size=2
remote.log.manager.task.retry.backoff.ms=500

# Log segment retention в remote storage (S3/GCS)
remote.log.storage.custom.storage.impl=io.confluent.kafka.storage.s3.S3RemoteStorageManager
s3.bucket.name=kafka-tiered-storage
s3.region=us-east-1

# Topic config
kafka-configs.sh --alter --topic logs --add-config \
  remote.storage.enable=true,local.retention.hours=24,remote.retention.hours=720

# Проверка
kafka-log-dirs.sh --describe --bootstrap-server localhost:9092
```

**Cost optimization**:
```yaml
# S3 Pricing vs EBS:
# EBS gp3: $0.08/GB-month
# S3 Standard: $0.023/GB-month
# S3 Glacier: $0.004/GB-month

# Паттерн:
# - Hot data (24h): local SSD (fast I/O)
# - Warm (30d): S3 Standard (analytics, replay)
# - Cold (>30d): S3 Glacier/IA (compliance, audit)

# Мониторинг cost
# - Отслеживай remote.log.manager.copy-lag
# - Алерт при high copy-lag = storage bottleneck
# - Регулярно очищай unused topics с `kafka-delete-records.sh`
```

**DevOps-контекст**:
- Tiered storage отделяет compute от storage. Позволяет хранить годы данных без раздувания кластера
- `local.retention` определяет, сколько держать на fast disk. Остальное tier'ится автоматически
- S3 lifecycle policies + Kafka retention = автоматический cost management
- Включай compression (`lz4`/`zstd`) до tiering. Снижает storage cost на 30-60%

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 25. Multi-protocol gateways: MQTT, AMQP 1.0, STOMP

**Задача**: Настроить мосты для legacy/IoT клиентов.

**Решение**:
```bash
# RabbitMQ: MQTT plugin
rabbitmq-plugins enable rabbitmq_mqtt
rabbitmq-plugins enable rabbitmq_web_mqtt
# rabbitmq.conf
mqtt.default_user = iot_device
mqtt.default_pass = iot_secret
mqtt.tcp_listen_options.backlog = 128

# Kafka: MQTT Connector (Confluent)
# Использует Kafka Connect MQTT source/sink
# Конфигурация через REST API Connect

# RabbitMQ: AMQP 1.0 plugin
rabbitmq-plugins enable rabbitmq_amqp1_0
# Поддержка JMS 2.0, .NET AMQP, Azure Service Bus compatibility

# Тестирование MQTT → RabbitMQ → Kafka
# 1. IoT device publishes to MQTT topic "sensors/temperature"
# 2. RabbitMQ route to exchange "iot.sensors"
# 3. Kafka Connect bridge to topic "raw-telemetry"
# 4. Streams process → enriched topics
```

**DevOps-контекст**:
- MQTT идеален для IoT (low bandwidth, unstable networks). QoS 0/1/2 = delivery guarantees
- AMQP 1.0 = enterprise стандарт. Совместим с Azure Service Bus, JMS, .NET
- Gateway pattern: legacy protocols → брокер → унифицированная обработка
- Мониторь connection limits и memory per connection. IoT devices = thousands of persistent connections

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

### 26. Cloud-native messaging: SNS, EventBridge, EventGrid patterns

**Задача**: Спроектировать event-driven архитектуру с managed сервисами.

**Решение**:
```yaml
# AWS SNS + SQS Fan-out
# SNS Topic: order-events
# Subscriptions: SQS (inventory), SQS (billing), Lambda (analytics), HTTP (webhook)
# Pattern: Pub/Sub without managing brokers

# AWS EventBridge (Complex routing)
# Event Bus: custom.orders
# Rule: {"source": ["com.myapp.orders"], "detail-type": ["created"]}
# Targets: SQS (queue), Step Functions (workflow), Lambda (validation)
# Schema Registry встроенный. Auto-discovery events

# Azure Event Grid
# Topic: orders
# Event Subscription: Storage Queue, Logic App, Service Bus
# Pattern: Serverless event routing, pay-per-event

# Сравнение:
# SQS: queue processing, decoupling
# SNS: fan-out, broadcasting
# EventBridge: complex routing, SaaS events, schema management
# Kafka: high-throughput, streaming, replay, stateful processing
```

**DevOps-контекст**:
- Managed брокеры = zero ops, но vendor lock-in и ограниченный контроль
- SNS + SQS = классический fan-out. EventBridge = modern event mesh с built-in routing
- Для replayability и complex transformations всё равно нужен Kafka/Kinesis
- Cost: SQS/SNS pay per request. Kafka pay per broker/GB. Считай TCO при >1M msg/day
- Integration pattern: Managed broker (ingest) → Kafka (stream processing) → Data Lake (analytics)

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

## 🧪 Чек-лист выполнения

- [ ] Установил и настроил RabbitMQ + Kafka (KRaft) в Docker
- [ ] Выполнил базовые pub/sub операции через CLI и код
- [ ] Настроил AWS SQS очереди, FIFO, DLQ routing
- [ ] Подключил приложение с безопасными connection strings
- [ ] Развернул кластер RabbitMQ (Quorum) и Kafka (ISR, replication)
- [ ] Настроил consumer groups, отслеживание lag, reset offsets
- [ ] Реализовал retention, compaction, persistence policies
- [ ] Подключил Prometheus + Grafana для мониторинга брокеров
- [ ] Настроил TLS, SASL, ACLs для security hardening
- [ ] Оптимизировал prefetch, batching, ack modes
- [ ] Развернул Kafka Connect + Schema Registry
- [ ] Настроил cross-cluster replication (MM2 / Shovel)
- [ ] Спроектировал partitioning strategy и autoscaling (KEDA)
- [ ] Реализовал Exactly-Once Semantics (или app-level idempotency)
- [ ] Провёл chaos tests: broker failure, network partition
- [ ] Настроил tiered storage и cost optimization
- [ ] Спроектировал cloud-native event mesh (SNS/EventBridge)

> [!TIP] Практика
> Для закрепления:
> 1. Разверни full event-driven stack: App → RabbitMQ/Kafka → Consumer → DB
> 2. Настрой KEDA autoscaling на основе lag в Kubernetes
> 3. Проведи load test: 100k msg/sec, замерь latency и throughput
> 4. Напиши runbook для DR: broker down, partition, data corruption
> 5. Интегрируй Schema Registry в CI/CD (validate schemas before deploy)

---

## 🔗 Связь с другими файлами

> [!TIP] Следующие шаги
> После выполнения практики:
> - [[Message Brokers теория]]: архитектура, паттерны, гарантии доставки, consensus
> - [[Message Brokers вопросы]]: подготовка к собеседованию
> - [[Databases практика]]: интеграция с БД, CDC, outbox pattern
> - [[CI_CD практика]]: event-driven pipelines, webhooks, async testing
> - [[Kubernetes практика]]: KEDA, Strimzi, RabbitMQ Operator, Helm charts

```mermaid
graph LR
    A[Message Brokers практика] --> B[[Message Brokers теория]]
    A --> C[[Message Brokers вопросы]]
    A --> D[[Databases практика]]
    A --> E[[Kubernetes практика]]
    B --> C
    style A fill:#336791,stroke:#333,stroke-width:2px
```

[[#🗂️ Оглавление для навигации|↑ К оглавлению]]

---

**Структура проекта**:
```
DevOps_start-main
├── 00_Fundamentals
│   ├── Linux
│   ├── Networking
│   └── Scripting
├── 01_Version_Control
│   └── Git
├── 02_Containers
│   ├── Docker
│   └── Kubernetes
├── 03_Infrastructure
│   ├── Terraform
│   ├── Ansible
│   └── AWS_Cloud
├── 04_CI_CD
│   ├── CI_CD
│   └── GitOps
├── 05_Observability
│   ├── Prometheus
│   ├── Grafana
│   ├── Loki
│   └── Tempo
├── 06_Databases
├── 07_Security
├── 08_Advanced
│   └── Message Brokers
│       ├── [[Message Brokers практика]] ← этот файл
│       ├── [[Message Brokers теория]]
│       └── [[Message Brokers вопросы]]
└── Roadmap
```

> [!NOTE] Выбор брокера
> | Сценарий | Рекомендуемый брокер |
> |----------|---------------------|
> | Simple decoupling, async tasks | RabbitMQ / SQS |
> | High throughput, streaming, replay | Kafka |
> | Serverless, event routing | EventBridge / SNS |
> | IoT, low bandwidth | MQTT + RabbitMQ |
> | Financial, exactly-once | Kafka (transactions) |
> 
> Для практики: начни с RabbitMQ (проще) → Kafka (streaming) → SQS (cloud-native).
