---
layout: post
title: kafka 日志版本变迁
categories: GitHub
description: 使用这个博客模板的朋友们时不时会提出一些问题，我将它们的解决方案逐渐整理归纳，汇总到这一篇帖子里。
keywords: kafka
topmost: true
---

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
3. [从老版本升级kafka](https://www.orchome.com/505)