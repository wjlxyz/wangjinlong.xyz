---
title: Kafka系列1：Kafka概况
categories:
    - '技术'
    - 'Kafka'
tags:
    - Kafka
---

Kafka是当前分布式系统中最流行的消息中间件之一，凭借着其高吞吐量的设计，在日志收集系统和消息系统的应用场景中深得开发者喜爱。
本篇就聊聊Kafka相关的一些知识点。主要包括以下内容：
- Kafka简介
    - Kafka特点
    - Kafka基本概念
    - Kafka架构
- Kafka的几个核心概念
    - 分区Partition
    - 复制Replication
    - 消息发送
    - 消费者组
    - 消费偏移量
- Kafka的工程应用

<!--more-->

### Kafka简介
#### Kafka特点
Kafka是最初由Linkedin公司开发，是一个分布式、分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统（也可以当做MQ系统），常见可以用于web/nginx日志、访问日志，消息服务等等，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。
相比于其他的消息队列中间件，Kafka的主要设计目标，也即其特点如下：
1. 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能。
2. 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条消息的传输。
3. 支持Kafka Server间的消息分区，及分布式消费，同时保证每个partition内的消息顺序传输。
4. 同时支持离线数据处理和实时数据处理。
5. Scale out:支持在线水平扩展
#### Kafka基本概念
**Broker**
- Kaka集群中的一台或多台服务器称为Broker。Broker存储Topic的数据。
- 如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个partition。
- 如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一个partition，剩下的M个broker不存储该topic的partition数据。
- 如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致Kafka集群数据不均衡。
**Topic**
- 发布到Kafka的每条消息都有一个类别，是个逻辑概念。
- 物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上，但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处
**Partition**
- 物理上的Topic分区，一个Topic可以分为多个Partition，至少有一个Partition。
- 每个Partition中的数据使用多个segment文件存储，每个Partition都是一个有序的队列，不同Partition间的数据是无序的。
- Partition中的每条消息都会被分配一个有序的ID（即offset）。
**Producer**
- 消息和数据的生产者。Producer将消息发布到Kafka的topic中。
- Broker接收到Producer发布的消息后，Broker将该消息追加到当前用于追加数据的segment文件中。
- Producer发送的消息，存储到一个Partition中，Producer也可以指定数据存储的Partition。
**Consumer**
- 消息和数据的消费者。Consumer从Broker中读取数据。
- Consumer可以消费多个topic中的数据。
**Consumer Group**
- 每个消费者都属于一个特定的消费者组。
- 可为每个Consumer指定group name，若不指定group name则属于默认的group。
- 一个Topic可以有多个消费者组，Topic的消息会被复制到所有的消费者组中，但每个消费者组只会把消息发送给该组中的一个消费者。
- 消费者组是Kafka用来实现一个Topic消息的广播和单播的手段。
**Leader**
- 每个Partition有多个副本，其中有且仅有一个作为leader。
- Leader是当前负责数据的读写的Partition。
**Follower**
- Follower跟随Leader，所有写请求都通过Leader路由，数据变更会广播给所有Follower，Follower与Leader保持数据同步。
- 如果Leader失效，则从Follower中选举出一个新的Leader。
- 如果Follower与Leader挂掉、卡住或同步太慢，Leader会把这个Follower从"in sync replicas"## 高吞吐量的分布式消息组件Kafka是如何工作的

Kafka是当前分布式系统中最流行的消息中间件之一，凭借着其高吞吐量的设计，在日志收集系统和消息系统的应用场景中深得开发者喜爱。
本篇就聊聊Kafka相关的一些知识点。主要包括以下内容：

- Kafka简介
	* Kafka特点
	* Kafka基本概念
	* Kafka架构

- Kafka的几个核心概念
	* 分区Partition
	* 复制Replication
	* 消息发送
	* 消费者组
	* 消费偏移量

- Kafka的工程应用

### Kafka简介

#### Kafka特点

Kafka是最初由Linkedin公司开发，是一个分布式、分区的、多副本的、多订阅者，基于zookeeper协调的分布式日志系统（也可以当做MQ系统），常见可以用于web/nginx日志、访问日志，消息服务等等，Linkedin于2010年贡献给了Apache基金会并成为顶级开源项目。
相比于其他的消息队列中间件，Kafka的主要设计目标，也即其特点如下：

1. 以时间复杂度为O(1)的方式提供消息持久化能力，即使对TB级以上数据也能保证常数时间的访问性能。
2. 高吞吐率。即使在非常廉价的商用机器上也能做到单机支持每秒100K条消息的传输。
3. 支持Kafka Server间的消息分区，及分布式消费，同时保证每个partition内的消息顺序传输。
4. 同时支持离线数据处理和实时数据处理。
5. Scale out:支持在线水平扩展

#### Kafka基本概念

**Broker**

- Kaka集群中的一台或多台服务器称为Broker。Broker存储Topic的数据。
- 如果某topic有N个partition，集群有N个broker，那么每个broker存储该topic的一个partition。
- 如果某topic有N个partition，集群有(N+M)个broker，那么其中有N个broker存储该topic的一个partition，剩下的M个broker不存储该topic的partition数据。
- 如果某topic有N个partition，集群中broker数目少于N个，那么一个broker存储该topic的一个或多个partition。在实际生产环境中，尽量避免这种情况的发生，这种情况容易导致Kafka集群数据不均衡。

**Topic**

- 发布到Kafka的每条消息都有一个类别，是个逻辑概念。
- 物理上不同Topic的消息分开存储，逻辑上一个Topic的消息虽然保存于一个或多个broker上，但用户只需指定消息的Topic即可生产或消费数据而不必关心数据存于何处

**Partition**

- 物理上的Topic分区，一个Topic可以分为多个Partition，至少有一个Partition。
- 每个Partition中的数据使用多个segment文件存储，每个Partition都是一个有序的队列，不同Partition间的数据是无序的。
- Partition中的每条消息都会被分配一个有序的ID（即offset）。

**Producer**

- 消息和数据的生产者。Producer将消息发布到Kafka的topic中。
- Broker接收到Producer发布的消息后，Broker将该消息追加到当前用于追加数据的segment文件中。
- Producer发送的消息，存储到一个Partition中，Producer也可以指定数据存储的Partition。

**Consumer**

- 消息和数据的消费者。Consumer从Broker中读取数据。
- Consumer可以消费多个topic中的数据。

**Consumer Group**

- 每个消费者都属于一个特定的消费者组。
- 可为每个Consumer指定group name，若不指定group name则属于默认的group。
- 一个Topic可以有多个消费者组，Topic的消息会被复制到所有的消费者组中，但每个消费者组只会把消息发送给该组中的一个消费者。
- 消费者组是Kafka用来实现一个Topic消息的广播和单播的手段。

**Leader**

- 每个Partition有多个副本，其中有且仅有一个作为leader。
- Leader是当前负责数据的读写的Partition。

**Follower**

- Follower跟随Leader，所有写请求都通过Leader路由，数据变更会广播给所有Follower，Follower与Leader保持数据同步。
- 如果Leader失效，则从Follower中选举出一个新的Leader。
- 如果Follower与Leader挂掉、卡住或同步太慢，Leader会把这个Follower从"in sync replicas"列表中删除，重新创建一个Follower。
#### Kafka架构
Kafka一般以集群方式来部署，一个典型的Kafka集群架构如下图所示：
![Kafka架构图](/pictures/kafka架构图.jpg)
###Kafka的几个核心概念
#### 分区Partition
**分区的几个特点**
- 分区是Kafka的基本存储单元，在一个Topic中会有一个或多个Partition，不同的Partition可位于不同的服务器节点上，物理上一个Partition对应于一个文件夹。
- Partition内包含一个或多个Segment，每个Segment又包含一个数据文件和一个与之对应的索引文件。
- 对于写操作，每次只会写Partition内的一个Segment；对于读操作，也只会顺序读取同一个Partition内的不同Segment。
- 逻辑上，可以把Partition当做一个非常长的数组，使用时通过这个数组的索引（offset）访问数据。
**高吞吐量设计**
分区正是Kafka高吞吐量设计的方法之一，具体体现在这样几点：
- 由于不同的Partition可位于不同的机器上，因此可以实现机器间的并行处理。
- 由于一个Partition对应一个文件夹，多个Partition也可位于同一台服务器上，这样就可以在同一台服务器上使不同的Partition对应不同的磁盘，实现磁盘间的并行处理。
- 故一般通过增加Partition的数量来提高系统的并行吞吐量，但也会增加轻微的延迟。
但以下这几种情况需要注意：
- 当一个Topic有多个消费者时，一个消息只会被一个消费者组里的一个消费者消费；
- 由于消息是以Partition为单位分配的，在不考虑Rebalance时，同一个Partition的数据只会被一个消费者消费，所以如果消费者的数量多于Partition的数量，就会存在部分消费者不能消费该Topic的情况，此时再增加消费者并不能提高系统的吞吐量；
- 在生产者和Broker的角度，对不同Partition的写操作是完全并行的，可是对于消费者其并发数则取决于Partition的数量。实际中配置的Partition数量需要根据所设计的系统吞吐量来推算。
#### 复制
**复制原理**
Kafka利用zookeeper来维护集群成员的信息，每个Broker实例都会被设置一个唯一的标识符，Broker在启动时会通过创建临时节点的方式把自己的唯一标识注册到zookeeper中，Kafka中的其他组件会监视Zookeeper里的/broker/ids路径，所以当集群中有Broker加入或退出时，其他组件就会收到通知。
集群间数据的复制机制，在Kafka中是通过Zookeeper提供的leader选举方式实现数据复制方案。
基本原理是：首先选举出一个leader，其他副本作为Follower，所有的写操作都先发给leader，然后再由leader把消息发给Follower。
复制功能是Kafka架构的核心之一，因为它可以在个别节点不可用时还能保证Kafka整体的可用性。
Kafka中的复制操作也是针对分区的。一个分区有多个副本，副本被保存在Broker上，每个Broker都可以保存上千个属于不同Topic和分区的副本。
副本有两种类型：
- leader副本：每个分区都会有，所有生产者和消费者的请求都会经过leader；
- follower副本：不处理客户端的请求，它的职责是从leader处复制消息数据，使自己和leader的状态保持一致；
- 如果leader节点宕机，那么某个follower就会被选为leader继续对外提供服务；
- 复制因子：一个分区有几个副本。
#### 消息发送方式
从生产者的角度来看，消息发送到Broker有三种方式：
- 立即发送：只发送消息，不关心消息发送的结果。本质上也是一种异步发送的方式，消息先存储在缓冲区中，达到设定条件后批量发送。当然这是kafka吞吐量最高的一种方式,并配合参数acks=0，这样生产者不需要等待服务器的响应，以网络能支持的最大速度发送消息。但是也是消息最不可靠的一种方式，因为对于发送失败的消息没有做任何处理。
- 同步发送：生产者发送消息后获取返回的Future对象，根据该对象的结果查看发送是否成功。如果业务要求消息必须是按顺序发送的，那么可以使用同步的方式，并且只能在一个partation上，结合参数设置retries的值让发送失败时重试，设置max_in_flight_requests_per_connection=1，可以控制生产者在收到服务器晌应之前只能发送1个消息，在消息发送成功后立刻flush，从而控制消息顺序发送。
- 异步发送：生产者发送消息时将注册的回调函数作为入参传入，生产者接收到Kafka服务器的响应时会触发执行回调函数。如果业务需要知道消息发送是否成功，并且对消息的顺序不关心，那么可以用异步+回调的方式来发送消息，配合参数retries=0，并将发送失败的消息记录到日志文件中。
#### 消息发送确认
消息发送到Broker后怎么算投递成功呢，Kafka有三种确认模式：
- 不等Broker确认就认为投递成功；
- 由leader来确认投递成功；
- 由所有的leader和follower都确认才认为是成功的。
三种模式对比的话，性能依次降低，但可靠性依次提高。
#### 消息重发机制
当从Broker接收到的是临时可恢复的异常时，生产者会向Broker重发消息，重发次数的限制值由初始化生产者对象的retries属性决定，在默认情况下生产者会在重试后等待100ms，可以通过retry.backoff.ms属性进行修改。
#### 批次发送
当有多条消息要被发送到同一个分区时，生产者会把它们放到同一个批次里，Kafka通过批次的概念来提高吞吐量，但同时也会增加延迟。
对批次的控制主要通过构建生产者对象时的两个属性来实现：
- batch.size：当发往每个分区的缓存消息数量达到这个数值时，就会触发一次网络请求，批次里的所有消息都会被发送出去；
- linger.ms：每条消息在缓存中的最长时间，如果超过这个时间就会忽略batch.size的限制，由客户端立即把消息发送出去。

### 消费者组
消费者组是Kafka提供的可扩展且具有容错性的消费机制，在一个消费者组内可以有多个消费者，它们共享一个唯一标识，即分组ID。组内的所有消费者协调消费它们订阅的主题下的所有分区的消息，但一个分区只能由同一个消费者组里的一个消费者来消费。
#### 广播和单播
一个Topic可以有多个消费者组，Topic的消息会被复制到所有的消费者组中，但每个消费者组只会把消息发送给一个消费者组里的某一个消费者。
如果要实现广播，只需为每个消费者都分配一个单独的消费者组接口
如果要实现单播，则需要把所有的消费者都设置在同一个消费者组里
#### 再均衡
消费者组里有新消费者加入或者有消费者离开，分区所有权会从一个消费者转移到另一个消费者
再均衡协议规定了一个消费者组下的所有消费者如何达成一致来分配主题下的每个分区
触发再均衡的场景有三种：
- 一是消费者组内成员发生变更
- 二是订阅的主题数量发生表更
- 三是订阅主题的分区数量发生变更

### 消费偏移量
Kafka中有一个叫作_consumer_offset特殊主题用来保存消息在每个分区的偏移量，消费者每次消费时都会往这个主题中发送消息，消息包含每个分区的偏移量。
如果消费者一直处于运行状态，偏移量没什么作用；如果消费者崩溃或者有新的消费者加入消费者组从而触发再均衡操作，再均衡之后该分区的消费者若不是之前的那个，提交偏移量就有用了。
维护消息偏移量对于避免消息被重复消费和遗漏消费，确保消息的ExactlyOnce至关重要，以下是不同的提交偏移量的方式：
- 自动提交：Kafka默认会定期自动提交偏移量，提交的时间间隔默认是5秒。此方式会产生重复处理消息的问题；
- 手动提交：在进行手动提交之前需要先关闭消费者的自动提交配置，然后用commitSync方法来提交偏移量。处理完记录后由开发者确保调用了commitSync方法，来减少重复处理消息的数量，但可能降低消费者的吞吐量；
- 异步提交：使用commitASync方法来提交最后一个偏移量。消费者只管发送提交请求，而不需要等待Broker的立即回应。

### Kafka的工程应用
Kafka主要用于三种场景：
- 基于Kafka的用户行为数据采集
- 基于Kafka的日志收集
- 基于Kafka的流量削峰
#### 基于Kafka的用户行为数据采集
要获取必要的数据进行用户行为等的分析，需要这样几个步骤：
- 前端数据（埋点）上报
- 接收前端数据请求
- 后端通过Kafka消费消息，必要时落库
- 分析用户行为
#### 基于Kafka的日志收集
各个应用系统在输出日志时利用高吞吐量的Kafka作为数据缓冲平台，将日志统一输出到Kafka，再通过Kafka以统一接口服务的方式开放给各种消费者。
做统一日志平台的方案，收集重要系统的日志集中到Kafka中，然后再导入ElasticSearch、HDFS、Storm等具体日志数据的消费者中，用于进行实时搜索分析、离线统计、数据备份、大数据分析等。
#### 基于Kafka的流量削峰
为了让系统在大流量场景下仍然可用，可以在系统中的重点业务环节加入消息队列作为消息流的缓冲，从而避免短时间内产生的高流量带来的压垮整个应用的问题。


### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)




