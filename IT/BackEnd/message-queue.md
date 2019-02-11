# 消息队列

## kafka

### 基础操作

```shell
bin/kafka-topics.sh --create --zookeeper localhost:2181 --replication-factor 3 --partitions 1 --topic replicated-topic
```
> 创建名为 `replicated-topic`, 副本数为 3, 分区数(?)为 1 的 topic

```shell
bin/kafka-topics.sh --list --zookeeper localhost:2181
```
> 列出 topic

```shell
bin/kafka-topics.sh --describe --zookeeper localhost:2181 --topic my-replicated-topic
```
> 查看 topic 状态

```shell
bin/kafka-topics.sh --alter --zookeeper localhost:2181 --topic my-replicated-topic
```
> 修改 topic

```shell
bin/kafka-topics.sh --delete --zookeeper localhost:2181 --topic my-replicated-topic
```
> 删除 topic

```shell
bin/kafka-topics.sh --entity-type topics --entity-name my-replicated-topic --zookeeper localhost:2181 --alter [--add-config x=y/--delete-config x]
```
> 添加/删除一个配置项

```shell
bin/kafka-console-producer.sh --broker-list localhost:9092 --topic test
```
> 作为生产者发送信息到 `test` 的 topic

```shell
bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic test --from-beginning
```
> 作为消费者接收 `test` topic 的信息

#### kafka-connect

官方工具, 通过它来 自定义(?)逻辑 以便进行与外部系统(服务器?)的交互

#### kafka-streams

Kafka Streams 是用于构建实时关键应用程序和微服务的客户端库

TODO:

### 概念

- topic: 数据主题(data subject), 每个 topic 有多个独立的分区日志
- 每个分区(partition) 都是 有序 且 顺序不可变 的数据集
- offset: 每个独立分区中每一个记录都会分配一个分区中唯一的 `offset`
- 每个分区都有一台 server 作为 `leader`, 任意台 server 作为 `fllowers`
- broker: 每个 kafka 服务器即为一个 broker , 一个集群由多个 broker 组成; 一个 broker 可以容纳多个 topic
- 生产者(producer):
  -
- 消费者(consumer):
  - 通过 **消费组(consumer group)** 名称来标识
  - 如果所有的消费者实例在 **同一消费组** 中, 消息记录会 负载平衡 到每一个消费者实例 -- 同一条 topic 的一条消息, 只能被同一个 consumer group 内的唯一一个 consumer 消费 (但多个 comsumer group 可同时消费这一消息)
  - 如果 所有的消费者实例 在 **不同的消费组** 中, 每条消息记录会 **广播** 到所有的消费者进程 --> 低效?
  
### 保证

high-level kafka 给予以下保证
- 生产者发送到特定 topic partition 的消息将按照发送的顺序处理; 也就是说, 如果记录 M1 和记录 M2 由相同的生产者发送, 并先发送 M1 记录, 那么 M1 的偏移比 M2 小, 并在日志中较早出现
- 一个消费者实例按照日志中的顺序查看记录
- 对于具有 N 个副本的主题, 我们最多容忍 N-1 个服务器故障, 从而保证不会丢失任何提交到日志中的记录

### 功能性

#### 作为消息系统

#### 作为存储系统

#### 用作流处理

#### 批处理

### 优雅的关机

当服务器正常关闭:

- 它将所有日志同步到磁盘, 以避免在重新启动时需要进行任何日志恢复活动(即验证日志尾部的所有消息的校验和). 由于日志恢复需要时间, 所以从侧面加速了重新启动操作
- 它将在关闭之前将以该服务器为 leader 的任何分区迁移到其他副本. 这将使 leader 角色传递更快, 并将每个分区不可用的时间缩短到几毫秒
  - leader 迁移需要添加配置: `controlled.shotdown.enabe=true`
  
### Balancing leadership

默认下, 一个 broker 在进入分区后(无论是新加入的 broker 还是 重新加入的), 只是所有分区的跟随者, 即意味着它不会用于客户端的读取和写入

可通过以下方法在有 borker  加入时重新设置副本 leader:
- 命令行: `bin/kafka-preferred-replica-election.sh --zookeeper zk_host:port/chroot`
- 配置: `auto.leader.rebalance.enable=true`

### 垮机架均衡副本

TODO:

### 集群之间镜像复制

## nsq

## redis
