---
title:  Kafka 笔记
classes: wide
categories:
  - 2021-07
tags:
  - kafka
type: note
---

Kafka is a mesage broker.

设计理念 Dumb broker / Smart consumer, 与 rabbitMq 的 Smart broker / Dumb consumer 相反。

dumb broker 使kafka有更好的性能和扩展性，因为所有消息直接 append 到 commit log里。

dumb broker 也意味着常见的消息队列功能（如消息优先级）在 kafka 里实现会[有点困难](https://www.confluent.io/blog/prioritize-messages-in-kafka/)。

Kafka supports streaming, marketed as streaming processing platform.

流处理（数据流入 kafka 时可以进行处理），使得 kafka 成为大数据处理重要的平台。

ref: [https://www.youtube.com/watch?v=Ofzwakas9Ek&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=10](https://www.youtube.com/watch?v=Ofzwakas9Ek&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=10)

## Concepts

ref: [https://www.youtube.com/watch?v=Lgzn6M4DIog&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=8](https://www.youtube.com/watch?v=Lgzn6M4DIog&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=8)

### Topic

N 个 producer 可以向同一个 topic 发送消息，N 个 consumer 可以订阅同一个 topic，每个 consumer 收到同样的消息。

如果一个 consumer 的处理能力不足以匹配上producer，可以把多个 consumer 组成一个 group，协同处理消息。

### Zookeeper

kafka 依赖 zookeeper 来做一些关键操作（例如 leader 选举），以及存放一些重要的 metadata

### Node & Partition

当topic的写入速度超过了单节点的处理能力，可以把topic分割成多个 partition，分散到多个node节点并行处理。一个node节点可以存放多个partition。

![node_partitions](../../assets/images/2021/07/node_partitions.png "node partitions")

consumer group里的 consumer 分别从某几个 partition 里读取数据。

## Reliability

ref: [https://www.youtube.com/watch?v=zFUoowJaZe8&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=8&t=47s](https://www.youtube.com/watch?v=zFUoowJaZe8&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=8&t=47s)

### consumer failure

consumer group里的一个consumer挂了，kafka会把相应的partition分配给组里的其他consumer，其他cosnumer可以从上次处理的offset开始处理。

#### offset

消息分散到多个partition里，每个partition里的消息顺序单调递增。

![offset](../../assets/images/2021/07/offset.png "offset")

0.9 版本以后，offset信息存在内部的topic里 [https://stackoverflow.com/questions/41137281/offsets-stored-in-zookeeper-or-kafka](https://stackoverflow.com/questions/41137281/offsets-stored-in-zookeeper-or-kafka)

**offset 如何保存？假设client的读取速度4000/s，每秒需要保存4000次？**

有几种方式：

- auto-commit: 客户端每5s保存一次offset信息。pro: 性能好；cons：客户端挂了后，没有及时更新offset，消息有可能重复消费。

- sync-commit: 客户端每消费一个或一批数据后，保存offset信息。 cons: 影响消费速度。

- async-commit: 上述两种方式的权衡，继承了各自的优缺点。

### broker failure

ref: [https://www.youtube.com/watch?v=4n-f-cXhTv8&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=7&t=2s](https://www.youtube.com/watch?v=4n-f-cXhTv8&list=PLjNqVu1lCpIatt8KnZlmYVtmP84cpTl8p&index=7&t=2s)

每个partition leader负责写入，leader可以有多个replica，分散到多个node，replica从leader拉取数据。

![replication](../../assets/images/2021/07/replication.png "replication")


#### in-sync relicas(ISR)

`replica.lag.time.max.ms=10s` 如果slave节点过去10s没从master拉取数据，或者最新数据比master过期超过10s，则该replica没有 in-sync

如果 isr 为空，同时leader failed，则写入和消费报错。

`unclean.leader.election.enable=true` 可以把非isr选举为leader，会丢失一部分数据。

---


## 实践

### docker 方式启动（内部访问）

```
docker network create rmoff_kafka
docker run --network=rmoff_kafka --rm --detach --name zookeeper -e ZOOKEEPER_CLIENT_PORT=2181 confluentinc/cp-zookeeper:5.5.0
docker run --network=rmoff_kafka --rm --detach --name kafka \
           -p 9092:9092 \
           -e KAFKA_BROKER_ID=1 \
           -e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
           -e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092 \
           -e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
           confluentinc/cp-kafka:5.5.0
```

测试列出 metadata 成功：

```
cyuser@cyuser-Vostro-3470:~$ kafkacat -L -b 10.6.5.112:9092
```

测试内部发送消息 成功：

```
$ docker run --network=rmoff_kafka --rm --name python_kafka_test_client \
        --tty python_kafka_test_client kafka:9092
```

测试外部发送消息 失败：

```
python3 ~/Desktop/python_kafka_test_client.py 10.6.5.112:9092
```

失败原因：客户端根据 ADVERTISED_LISTENERS 配置的地址来连接broker，该地址需要外部可以访问

### docker 方式启动（内外部访问）

```
docker run --network=rmoff_kafka --rm --detach --name kafka \
-p 19092:19092 \
-p 19093:19093 \
-e KAFKA_BROKER_ID=1 \
-e KAFKA_ZOOKEEPER_CONNECT=zookeeper:2181 \
-e KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092,CONNECTIONS_FROM_HOST://10.6.5.112:19092,FOO://10.6.5.112:19093 \
-e KAFKA_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT,CONNECTIONS_FROM_HOST:PLAINTEXT,FOO:PLAINTEXT \
-e KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR=1 \
confluentinc/cp-kafka:5.5.0
```

测试内部发送消息 成功：

```
$ docker run --network=rmoff_kafka --rm --name python_kafka_test_client \
        --tty python_kafka_test_client kafka:9092
```

测试外部发送消息 成功：

```
python3 ~/Desktop/python_kafka_test_client.py 10.6.5.112:19092
```

### Listeners vs Advertised Listeners

listener 用来建立连接，传输metadata，advertised listener 用来跟客户端传输数据。类似 ftp 的两个端口，分别传输控制信息和数据。

### Console Producer & Consumer

```
export ZK_SERVER=10.6.5.112
export KAFKA_SERVER=10.6.5.112

#list & create & decribe topic

docker run --rm confluentinc/cp-kafka:5.5.0 /usr/bin/kafka-topics --list --zookeeper ${ZK_SERVER}:2181

docker run --rm confluentinc/cp-kafka:5.5.0 /usr/bin/kafka-topics --create --zookeeper ${ZK_SERVER}:2181 --partitions 1 --replication-factor 1 --topic test-console-topic

docker run --rm confluentinc/cp-kafka:5.5.0 /usr/bin/kafka-topics --describe --zookeeper ${ZK_SERVER}:2181 --topic test-console-topic

#start producer and consumer

docker run -ti --rm confluentinc/cp-kafka:5.5.0 /usr/bin/kafka-console-producer --bootstrap-server ${KAFKA_SERVER}:19092 --topic test-console-topic

docker run --rm confluentinc/cp-kafka:5.5.0 /usr/bin/kafka-console-consumer --bootstrap-server ${KAFKA_SERVER}:19092 --topic test-console-topic --from-beginning
```

## Examples(java)

[https://github.com/xiez/meetup-kafka](https://github.com/xiez/meetup-kafka)

## Readings

- [https://www.confluent.io/blog/kafka-client-cannot-connect-to-broker-on-aws-on-docker-etc/](https://www.confluent.io/blog/kafka-client-cannot-connect-to-broker-on-aws-on-docker-etc/)

- [https://www.confluent.io/blog/kafka-listeners-explained/](https://www.confluent.io/blog/kafka-listeners-explained/)
