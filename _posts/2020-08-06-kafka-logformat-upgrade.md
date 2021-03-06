---
layout: post
title: kafka 升级
categories: kafka
description:  kafka 升级步骤
keywords: kafka
# topmost: true
---

本文介绍如何升级kafka以及kafka各版本的日志格式

## kafka 升级步骤
1. 去线上机器kafka根目录下查询版本,得知旧版本号为 1.0.1
    ```shell
    $ find ./libs/ -name \*kafka_\* | head -1 | grep -o '\kafka[^\n]*'
    kafka_2.11-1.0.1-javadoc.jar
    ```
2. 在server.property中增加配置项，使用新版本（2.3.0）二进制文件在新机器上以此配置启动kafka broker进程
   ```shell
    inter.broker.protocol.version= 1.0.1
    log.message.format.version= 1.0.1
   ```
  
3. 将部分topic 迁移到新机器上 ，观察其影响一周。
4. 如果第3步 kafka新版本无异常表现良好，则将所有topic迁移到新机器上，停掉所有老机器上的broker
5. 修改server.property中的配置项,逐台以此配置重启所有broker
   ```
    inter.broker.protocol.version= 2.3.0
    log.message.format.version= 2.3.0
   ```  

##  kafka 新特性 static membership
> kafka 旧版本的成员关系可以理解为动态成员关系，新的静态成员关系本质是为了减少消费者组重平衡，待补充。

## kafka manager监控指标解释
1. Brokers Skewed%: Percentage of brokers having more partitions than the average 
    如果代理的分区数大于给定主题上每个代理的平均分区数，则代理就会发生倾斜。
    eg . 2 brokers、4 partitions , 如果有个分区为 3>  4 / 2,则 broker 就发生了倾斜  
2. Broker Spread% : Percentage of cluster brokers having partitions from the topic
    brokers spread  是集群中具有给定主题分区的代理的百分比。
    eg . 3个brokers共享2个partitions,因此存在66%的brokers有这个主题的分区
3. Under Replicated %: Percentage of partitions having a missing replica   不同步副本百分比
4. Brokers Leader Skew %: Percentage of brokers having more partitions as leader than the average

## kafka 至今共经历了三个版本变化

1. v0 (0.10.0以前)
2. v1 (0.10.0~0.11.0) 增加了 Timestamp和lz4压缩类型
3. v2 (0.11.0~2.2.3) 消息格式相交于以前发生了巨大变化，新增了（ProducerId，ProducerEpoch，Headers等）,编码方面（部分）也使用varint代替int等
* 目前log版本为0.10.1,如果将其升级到2.2.3 ，broker在发送响应到旧版本消费者之前转换到之前的格式,将会导致broker负载变大，为避免转换，需将日志格式保留。
* 如需升级到新版本，需先升级broker ，然后快速升级 producer和consumer发送消费新格式的消息

记录批处理的磁盘格式（0.10.0以前）
```
MessageSet (Version: 0) => [offset message_size message]
    offset => INT64
    message_size => INT32
    message => crc magic_byte attributes key value
        crc => INT32
        magic_byte => INT8
        attributes => INT8
            bit 0~2:
                0: no compression
                1: gzip
                2: snappy
            bit 3~7: unused
        key => BYTES
        value => BYTES
```
记录批处理的磁盘格式（0.10.0~0.11.x）
```
MessageSet (Version: 1) => [offset message_size message]
    offset => INT64
    message_size => INT32
    message => crc magic_byte attributes key value
        crc => INT32
        magic_byte => INT8
        attributes => INT8
            bit 0~2:
                0: no compression
                1: gzip
                2: snappy
                3: lz4
            bit 3: timestampType
                0: create time
                1: log append time
            bit 4~7: unused
        timestamp =>INT64
        key => BYTES
        value => BYTES
```
记录批处理的磁盘格式（0.11以后）
```
baseOffset: int64
batchLength: int32
partitionLeaderEpoch: int32
magic: int8 (current magic value is 2)
crc: int32
attributes: int16
    bit 0~2:
        0: no compression
        1: gzip
        2: snappy
        3: lz4
        4: zstd
    bit 3: timestampType
    bit 4: isTransactional (0 means not transactional)
    bit 5: isControlBatch (0 means not a control batch)
    bit 6~15: unused
lastOffsetDelta: int32
firstTimestamp: int64
maxTimestamp: int64
producerId: int64
producerEpoch: int16
baseSequence: int32
records: [Record]
Record：
length: varint
attributes: int8
    bit 0~7: unused
timestampDelta: varint
offsetDelta: varint
keyLength: varint
key: byte[]
valueLen: varint
value: byte[]
Headers => [Header]
header:
headerKeyLength: varint
headerKey: String
headerValueLength: varint
Value: byte[]
```


## 参考资料 
1. [messageformat](https://kafka.apache.org/documentation/#messageformat)
2. [A Guide To The Kafka Protocol](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol#AGuideToTheKafkaProtocol-Messagesets)
3. [kakfa静态成员关系](https://cwiki.apache.org/confluence/display/KAFKA/KIP-345%3A+Introduce+static+membership+protocol+to+reduce+consumer+rebalances)
4. [从老版本升级kafka](https://www.orchome.com/505)