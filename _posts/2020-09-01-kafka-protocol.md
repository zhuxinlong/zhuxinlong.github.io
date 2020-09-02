---
layout: post
title:  kafka协议指南
categories: kafka
description:  kafka协议指南
keywords: kafka protocol
# topmost: true
---

本文档涵盖了Kafka 0.8及更高版本中实现的协议。它的目的是提供一个可读的协议指南，包括可用的请求以及它们的二进制格式，以及使用它们实现client的正确方法。本文档假设您理解这里描述的基本设计和术语。

## 概述

Kafka协议相当简单，只有6个核心client请求api。

    1. Metadata: 描述当前可用的代理、它们的主机和端口信息，并给出关于哪个代理主机和哪个分区的信息。
    2. Send: 向代理发送消息
    3. Fetch: 从代理获取消息，获取数据，获取集群元数据，获取关于主题的偏移信息。
    4. Offsets: 获取关于给定主题分区的可用偏移量的信息。
    5. Offset Commit : 为消费者组提交一组偏移量
    6. Offset Fetch : 为消费者组获取一组偏移量

下面将详细描述其中的每一个。此外，从0.9开始，Kafka支持对消费者和Kafka Connect的一般组管理。客户端API由5个请求组成:

    1. GroupCoordinator : 定位一个组当前的协调器
    2. JoinGroup ： 成为一个组的成员，如果组没有活动成员就创建这个组。
    3. SyncGroup： 同步组中所有成员的状态(例如，将分区分配给使用者)。
    4. Heartbeat ： 保活一个组成员
    5. LeaveGroup  ： 直接离开组
   
最后，有几个管理api可用于监视/管理Kafka集群([KIP-4](https://cwiki.apache.org/confluence/display/KAFKA/KIP-4+-+Command+line+and+centralized+administrative+operations)完成后，这个列表将会增长)

    1. DescribeGroups :用于检查一组组的当前状态(例如，查看用户分区分配)。
    2. ListGroups : 列出由代理管理的当前组。


## 前言

### Network 
Kafka使用TCP上的二进制协议。协议将所有api定义为请求响应消息对。所有消息都有大小分隔符，由以下基本类型组成。

client端启动套接字连接，然后编写一系列请求消息并读取相应的响应消息。在连接或断开连接时不需要握手。如果您维护用于许多请求的持久连接，以摊销TCP握手的成本，TCP会更高兴。

client端可能需要维护到多个代理的连接，因为数据是分区的，client需要与拥有其数据的服务器通信。但是，通常不需要维护从单个客户机实例到单个代理的多个连接(即连接池)。

服务器保证在单个TCP连接上，请求将按发送顺序处理，响应也将按发送顺序返回。为了保证这种顺序，代理的请求处理只允许每个连接有一个正在运行的请求。请注意，客户端可以(理想情况下应该)使用非阻塞IO来实现请求流水线并实现更高的吞吐量。即。客户端甚至可以在等待前一个请求的响应时发送请求，因为未完成的请求将被缓冲在底层OS套接字缓冲区中。所有请求都是由客户机发起的，并从服务器生成相应的响应消息(除非特别说明)。

服务器对请求大小有一个可配置的最大限制，任何超过此限制的请求都会导致套接字断开连接。

### Partitioning and bootstrapping

Kafka 是一个分区系统，所以并不是所有的服务器都有完整的数据集。相反，请回忆一下，主题被划分为预定义的多个分区，并且每个分区都根据复制因子复制了多份。主题分区本身只是按“提交日志”编号的顺序排列 0,1,2...P。

有这类系统都有这样一个问题:如何将特定的数据段分配给特定的分区。Kafka客户机直接控制这个分配，代理本身不强制要求将消息发布到具体哪个分区。要发布消息，客户机需要指定分区，并且在获取消息时，从指定分区获取消息。如果两个客户机想使用相同的分区方案，它们必须使用相同的方法来计算键到分区的映射。

发布或获取数据的请求必须发送给对应分区的leader代理。此条件由代理强制执行，因此对错误代理的特定分区的请求将导致NotLeaderForPartition错误代码(如下所述)。

客户机如何查找存在哪些主题、它们有哪些分区以及哪些代理当前托管这些分区，以便将其请求定向到正确的主机?此信息是动态的，因此您不能仅使用一些静态映射文件配置每个客户机。相反，所有Kafka代理都可以响应描述集群当前状态的元数据请求:有哪些主题，这些主题有哪些分区，哪些代理是这些分区的领导者，以及这些代理的主机和端口信息。


换句话说，客户机需要以某种方式找到一个代理，该代理将告诉客户机存在的所有其他代理以及它们承载的分区。第一个代理本身可能会出现故障，因此客户机实现的最佳实践是从两个或三个broker url列表中启动。然后，用户可以选择使用负载均衡器，或者只是静态地在客户端配置两个或三个kafka主机。
客户端不需要保持轮询来查看集群是否已经更改;当实例化元数据缓存该元数据时，它可以获取一次元数据，直到它接收到指示元数据已过期的错误为止。
此错误可以以两种形式出现:(1)套接字错误，指示客户机不能与特定代理通信;(2)响应请求时的错误代码，指示此代理不再承载请求数据的分区。
    
    1. 循环遍历一个“引导”kafka url列表，直到找到一个可以连接的。获取集群元数据。
    2. 处理获取或生成请求，根据它们发送或获取的主题/分区将它们定向到适当的代理。
    3. 如果出现匹配的错误，请刷新元数据并重试。

### 分区策略

如上所述，将消息分配给分区是产生客户机控件的事情。也就是说，这个功能应该如何向最终用户公开?
在Kafka中，分区实际上有两个目的:
  
    1. 它通过代理平衡数据和请求负载
    2. 它作为一种方法，在允许本地状态和保持分区内秩序的同时，在使用者进程之间划分处理。我们称之为语义划分。

对于给定的用例，您可能只关心其中之一，或者两者都关心。
要实现简单的负载平衡，客户机只需在所有代理上轮询请求即可。另一种选择是，在生产者比代理多得多的环境中，让每个客户机随机选择一个分区并发布到该分区。第二种策略将导致更少的TCP连接。

语义分区意味着使用消息中的某个键将消息分配给分区。例如，如果您正在处理单击消息流，您可能希望按用户id对该流进行分区，以便特定用户的所有数据都将流向单个使用者。要实现这一点，客户机可以获取与消息关联的键，并使用该键的散列来选择要将消息传递到哪个分区。

### Batching 批量
为了提高效率，我们的api鼓励将小的东西打包在一起。我们发现这是一个非常重要的性能胜利。我们发送消息的API和获取消息的API总是处理一系列消息，而不是单个消息。

聪明的客户端可以利用这一点，并支持“异步”模式，在这种模式下，它将单独发送的消息进行批处理，然后以更大的块发送它们。我们更进一步，允许跨多个主题和分区进行批处理，因此一个generate请求可能包含要追加到多个分区的数据，而一个fetch请求可能同时从多个分区中提取数据。

客户端实现者可以选择忽略这一点，如果愿意，可以一次发送所有内容

### 版本和兼容性

该协议旨在以向后兼容的方式支持增量演化。我们的版本控制基于每个api，每个版本都包含一个请求和响应对。每个请求都包含一个API键，该键标识被调用的API，以及一个版本号，该版本号指示请求的格式和预期的响应格式。

其目的是让客户端实现协议的特定版本，并在其请求中指明该版本。我们的目标主要是允许API在不允许停机且不能同时更改所有客户机和服务器的环境中进行改进。

服务器将拒绝其不支持的版本的请求，并将始终根据客户端请求中包含的版本以其期望的协议格式响应客户端。预期的升级路径是首先在服务器上推出新特性(老客户机不使用它们)，然后随着新客户机的部署，这些新特性将逐渐得到利用。

目前，所有版本的基线都是0，随着我们对这些api的改进，我们将分别指出每个版本的格式。

## 协议

###  基本数据类型
协议是由以下基本类型构建的：

__定长基本类型__

int8、int16、int32、int64—具有给定精度(以位为单位)的带符号整数，以大端字节序存储

__变长基本类型__

字节，字符串——这些类型由一个带符号的整数组成，它的长度为N，后面跟着N个字节的内容。长度为-1表示null。字符串的大小使用int16，字节使用int32

__Arrays__

这是处理重复结构的符号。它们总是被编码为一个int32大小，包含长度N，然后是结构的N次重复，结构本身可以由其他基本类型组成。在下面的BNF语法中，我们将把结构foo的数组显示为[foo]。

### 关于阅读请求格式语法的说明

下面的BNFs为请求和响应二进制格式提供了一个精确的上下文自由语法。

对于每个API，我将同时给出请求和响应，然后给出所有子定义。
BNF故意不紧凑，以便提供人类可读的名称(例如，我为ErrorCode定义了一个产品，尽管它只是一个int16，以便给它一个符号名称)。
与BNF中通常一样，一个结果序列表示连接，因此下面给出的MetadataRequest将是一个字节序列，首先包含一个VersionId，然后是一个ClientId，然后是一个主题名称数组(每个主题名称都有自己的定义)。

结果总是用驼峰格式给出，原始类型用小写字母给出。当有多个可能的结果时，这些结果用`|`分隔，并可以用圆括号括起来进行分组。顶级定义总是先给出，随后的子部分缩进。


### 常见的请求和响应结构
所有的请求和响应都来自以下语法，这些语法将在本文档的其余部分逐步描述:

```bnf
RequestOrResponse => Size (RequestMessage | ResponseMessage)
  Size => int32
```

Field|Description
---------|----------
MessageSize|  MessageSize字段以字节为单位给出后续请求或响应消息的大小。客户机可以读取请求，首先将这个4字节大小读取为整数N，然后读取和解析请求的后续N字节



__Requests__

所有的请求都有以下格式:
```
RequestMessage => ApiKey ApiVersion CorrelationId ClientId RequestMessage
  ApiKey => int16
  ApiVersion => int16
  CorrelationId => int32
  ClientId => string
  RequestMessage => MetadataRequest | ProduceRequest | FetchRequest | OffsetRequest | OffsetCommitRequest | OffsetFetchRequest

```

Field|Description
---------|----------
 ApiKey|  这是被调用API的数字id(即它是元数据请求、生成请求、获取请求等)
 ApiVersion| 这是这个api的数字版本号。我们为每个API提供版本，这个版本号允许服务器随着协议的发展正确地解释请求。响应的格式始终与请求版本对应。
 CorrelationId|这是一个用户提供的整数。它将在响应中由服务器返回，未修改。它对于在客户机和服务器之间匹配请求和响应非常有用。
 ClientId | 这是为客户机应用程序提供的用户标识符。用户可以使用任何他们喜欢的标识符，它将用于记录错误、监视聚合等。例如，可能不仅要监视每秒的请求总数，还要监视来自每个客户机应用程序的请求数(每个客户机应用程序可以驻留在多个服务器上)。此id作为跨来自特定客户机的所有请求的逻辑分组

 下面将描述各种请求和响应消息。

 __Responses__

 ```
Response => CorrelationId ResponseMessage
CorrelationId => int32
ResponseMessage => MetadataResponse | ProduceResponse | FetchResponse | OffsetResponse | OffsetCommitResponse | OffsetFetchResponse
 ```
响应总是匹配成对的请求(例如，我们将发送一个MetadataResponse作为对MetadataRequest的返回)。

__Message sets__

产生和获取请求的一个共同结构是消息集格式。kafka中的消息是具有少量关联元数据的键值对。消息集就是包含偏移量和大小信息的消息序列。这种格式恰好同时用于代理上的磁盘存储和联机格式。

消息集也是Kafka中的压缩单元，我们允许消息递归地包含压缩的消息集，从而允许批量压缩。

注意:与协议中的其他数组元素不同，messageset的前面不带int32。

```
MessageSet => [Offset MessageSize Message]
  Offset => int64
  MessageSize => int32
```

```
v0
Message => Crc MagicByte Attributes Key Value
  Crc => int32
  MagicByte => int8
  Attributes => int8
  Key => bytes
  Value => bytes
  
v1 (supported since 0.10.0)
Message => Crc MagicByte Attributes Key Value
  Crc => int32
  MagicByte => int8
  Attributes => int8
  Timestamp => int64
  Key => bytes
  Value => bytes
```


在Kafka 0.11中，“MessageSet”和“Message”的结构发生了显著变化。不仅添加了新字段来支持新特性，比如只支持一次语义和记录头，而且取消了以前版本的消息格式的递归特性，采用了扁平结构。“MessageSet”现在称为“RecordBatch”，它包含一个或多个“记录”(而不是“消息”)。启用压缩时，RecordBatch头仍然未压缩，但是将记录压缩在一起。此外，“Record”中的多个字段是varint编码的，这将为较大的批节省大量空间。

```
RecordBatch =>
  FirstOffset => int64
  Length => int32
  PartitionLeaderEpoch => int32
  Magic => int8 
  CRC => int32
  Attributes => int16
  LastOffsetDelta => int32
  FirstTimestamp => int64
  MaxTimestamp => int64
  ProducerId => int64
  ProducerEpoch => int16
  FirstSequence => int32
  Records => [Record]
  
Record =>
  Length => varint
  Attributes => int8
  TimestampDelta => varint
  OffsetDelta => varint
  KeyLen => varint
  Key => data
  ValueLen => varint
  Value => data
  Headers => [Header]
  
Header => HeaderKey HeaderVal
  HeaderKeyLen => varint
  HeaderKey => string
  HeaderValueLen => varint
  HeaderValue => data
```



### The APIs
本节详细介绍每个api、它们的用法、二进制格式以及它们字段的含义

#### Metadata API

这个API回答了以下问题:

* 存在哪些topic?
* 每个主题有多少个分区?
* 当前哪个代理是每个分区的领导者?
* 每个代理的主机和端口是什么?

这是惟一可以向集群中的任何代理发出的请求。

由于可能有许多主题，客户端可以提供一个可选的主题名称列表，以便只返回主题子集的元数据。

返回的元数据位于分区级别，但为了方便和避免冗余，按主题分组。对于每个分区，元数据包含领导者的信息，以及所有副本的信息，以及当前同步的副本列表。

注意:如果“auto.create.topics。在代理配置中设置“enable”，主题元数据请求将创建具有默认复制因子和分区数的主题。

```

TopicMetadataRequest => [TopicName]
  TopicName => string
```
TopicName:要为其生成元数据的主题。如果为空，则请求将生成所有主题的元数据。

```

MetadataResponse => [Broker][TopicMetadata]
  Broker => NodeId Host Port  (any number of brokers may be returned)
    NodeId => int32
    Host => string
    Port => int32
  TopicMetadata => TopicErrorCode TopicName [PartitionMetadata]
    TopicErrorCode => int16
  PartitionMetadata => PartitionErrorCode PartitionId Leader Replicas Isr
    PartitionErrorCode => int16
    PartitionId => int32
    Leader => int32
    Replicas => [int32]
    Isr => [int32]
```

## 参考资料
1. [kafka协议指南](https://cwiki.apache.org/confluence/display/KAFKA/A+Guide+To+The+Kafka+Protocol)