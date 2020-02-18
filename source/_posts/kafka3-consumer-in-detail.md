---
title: Kafka系列3：深入理解Kafka消费者
categories:
    - '技术'
    - 'Kafka'
tags:
    - Kafka
---

上面两篇聊了Kafka概况和Kafka生产者，包含了Kafka的基本概念、设计原理、设计核心以及生产者的核心原理。本篇单独聊聊Kafka的消费者，包括如下内容：
- 消费者和消费者组
- 如何创建消费者
- 如何消费消息
- 消费者配置
- 提交和偏移量
- 再均衡
- 结束消费

<!--more-->

### 消费者和消费者组
#### 概念
**Kafka消费者**对象订阅主题并接收Kafka的消息，然后验证消息并保存结果。
**Kafka消费者**是**消费者组**的一部分。一个消费者组里的消费者订阅的是同一个主题，每个消费者接收主题一部分分区的消息。
消费者组的设计是对消费者进行的一个横向伸缩，用于解决消费者消费数据的速度跟不上生产者生产数据的速度的问题，通过增加消费者，让它们分担负载，分别处理部分分区的消息。
#### 消费者数目与分区数目
在一个消费者组中的消费者消费的是一个主题的部分分区的消息，而一个主题中包含若干个分区，一个消费者组中也包含着若干个消费者。当二者的数量关系处于不同的大小关系时，Kafka消费者的工作状态也是不同的。看以下三种情况：
1. 消费者数目<分区数目：此时不同分区的消息会被均衡地分配到这些消费者；
2. 消费者数目=分区数目：每个消费者会负责一个分区的消息进行消费；
3. 消费者数目>分区数目：此时会有多余的消费者处于空闲状态，其他的消费者与分区一对一地进行消费。
#### 分区再均衡
当消费者数目与分区数目在以上三种关系间变化时，比如有新的消费者加入、或者有一个消费者发生崩溃时，会发生**分区再均衡**。
**分区再均衡**是指分区的所有权从一个消费者转移到另一个消费者。再均衡为消费者组带来了高可用性和伸缩性。但是同时，也会发生如下问题：
- 在再均衡发生的时候，消费者无法读取消息，会造成整个消费者组有一小段时间的不可用；
- 当分区被重新分配给另一个消费者时，消费者当前的读取状态会丢失，它有可能需要去刷新缓存，在它重新恢复状态之前会拖慢应用。
因此也要尽量避免不必要的再均衡。
那么消费者组是怎么知道一个消费者可不可用呢？
消费者通过向被指派为群组协调器的Broker发送心跳来维持它们和群组的从属关系以及它们对分区的所有权关系。只要消费者以正常的时间间隔发送心跳，就被认为是活跃的，说明它还在读取分区里的消息。消费者会在轮询消息或提交偏移量时发送心跳。如果消费者停止发送心跳的时间足够长，会话就会过期，群组协调器认为它已经死亡，就会触发一次再均衡。
还有一点需要注意的是，当发生再均衡时，需要做一些清理工作，具体的操作方法可以通过在调用subscribe()方法时传入一个ConsumerRebalanceListener实例即可。
### 如何创建消费者
创建Kafka的消费者对象的过程与创建生产者的过程是类似的，需要传入必要的属性。在创建消费者的时候以下以下三个选项是必选的：
- bootstrap.servers ：指定 broker 的地址清单，清单里不需要包含所有的 broker 地址，生产者会从给定的 broker 里查找 broker 的信息。不过建议至少要提供两个 broker 的信息作为容错；
- key.deserializer ：指定键的反序列化器；
- value.deserializer ：指定值的反序列化器。
后两个序列化器的说明与生产者的是一样的。
一个简单的创建消费者的代码样例如下：
```java
String topic = "Hello";
String group = "group1";
Properties props = new Properties();
props.put("bootstrap.servers", "server:9091");
/*指定分组 ID*/
props.put("group.id", group);
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
```
### 如何消费消息
#### 订阅主题
创建了Kafka消费者之后，接着就可以订阅主题了。订阅主题可以使用如下两个 API :
- consumer.subscribe(Collection<String> topics) ：指明需要订阅的主题的集合；
- consumer.subscribe(Pattern pattern) ：使用正则来匹配需要订阅的集合。
代码样例：
```java
consumer.subscribe(Collections.singletonList(topic));
```
#### 轮询消费
消息轮询是消费者API的核心，消费者通过轮询 API(poll) 向服务器定时请求数据。一旦消费者订阅了主题，轮询就会处理所有的细节，包括群组协调、分区再均衡、发送心跳和获取数据，这使得开发者只需要关注从分区返回的数据，然后进行业务处理。
一个简单的消费者消费的代码样例如下：
```java
try {
    while (true) {
        // 轮询获取数据
        ConsumerRecords<String, String> records = consumer.poll(Duration.of(100, ChronoUnit.MILLIS));
        for (ConsumerRecord<String, String> record : records) {
            System.out.printf("topic = %s,partition = %d, key = %s, value = %s, offset = %d,\n",
           record.topic(), record.partition(), record.key(), record.value(), record.offset());
        }
    }
} finally {
    consumer.close();
}
```
### 消费者配置
与生产者类似，消费者也有完整的配置列表。接下来一一介绍这些重要的属性。
#### fetch.min.byte
消费者从服务器获取记录的最小字节数。如果可用的数据量小于设置值，broker 会等待有足够的可用数据时才会把它返回给消费者。主要是为了降低消费者和Broker的工作负载。
#### fetch.max.wait.ms
broker 返回给消费者数据的等待时间，默认是 500ms。如果消费者获取最小数据量的要求得不到满足，就会在等待最多该属性所设置的时间后获取到数据。实际要看二者哪个条件先满足。
#### max.partition.fetch.bytes
该属性指定了服务器从每个分区返回给消费者的最大字节数，默认为 1MB。
#### session.timeout.ms
消费者在被认为死亡之前可以与服务器断开连接的时间，默认是 3s。
#### auto.offset.reset
该属性指定了消费者在读取一个没有偏移量的分区或者偏移量无效的情况下该作何处理：
- latest (默认值) ：在偏移量无效的情况下，消费者将从最新的记录开始读取数据（在消费者启动之后生成的最新记录）;
- earliest ：在偏移量无效的情况下，消费者将从起始位置读取分区的记录。
#### enable.auto.commit
是否自动提交偏移量，默认值是 true。为了避免出现重复消费和数据丢失，可以把它设置为 false。
#### client.id
客户端 id，服务器用来识别消息的来源。
#### max.poll.records
单次调用 poll() 方法能够返回的记录数量。
#### receive.buffer.bytes & send.buffer.byte
这两个参数分别指定 TCP socket 接收和发送数据包缓冲区的大小，-1 代表使用操作系统的默认值。
### 提交和偏移量
**提交**是指更新分区当前位置的操作，分区当前的位置，也就是所谓的**偏移量**。
#### 什么是偏移量
Kafka 的每一条消息都有一个偏移量属性，记录了其在分区中的位置，偏移量是一个单调递增的整数。消费者通过往一个叫作 ＿consumer_offset 的特殊主题发送消息，消息里包含每个分区的偏移量。 如果消费者一直处于运行状态，那么偏移量就没有 什么用处。不过，如果有消费者退出或者新分区加入，此时就会触发再均衡。完成再均衡之后，每个消费者可能分配到新的分区，而不是之前处理的那个。为了能够继续之前的工作，消费者需要读取每个分区最后一次提交的偏移量，然后从偏移量指定的地方继续处理。 因为这个原因，所以如果不能正确提交偏移量，就可能会导致数据丢失或者重复出现消费，比如下面情况：
- 如果提交的偏移量小于客户端处理的最后一个消息的偏移量 ，那么处于两个偏移量之间的消息就会被重复消费；
- 如果提交的偏移量大于客户端处理的最后一个消息的偏移量，那么处于两个偏移量之间的消息将会丢失。
#### 偏移量提交
那么消费者如何提交偏移量呢？
Kafka 支持自动提交和手动提交偏移量两种方式。
##### 自动提交：
只需要将消费者的 enable.auto.commit 属性配置为 true 即可完成自动提交的配置。 此时每隔固定的时间，消费者就会把 poll() 方法接收到的最大偏移量进行提交，提交间隔由 auto.commit.interval.ms 属性进行配置，默认值是 5s。
使用自动提交是存在隐患的，假设我们使用默认的 5s 提交时间间隔，在最近一次提交之后的 3s 发生了再均衡，再均衡之后，消费者从最后一次提交的偏移量位置开始读取消息。这个时候偏移量已经落后了 3s ，所以在这 3s 内到达的消息会被重复处理。可以通过修改提交时间间隔来更频繁地提交偏移量，减小可能出现重复消息的时间窗，不过这种情况是无法完全避免的。基于这个原因，Kafka 也提供了手动提交偏移量的 API，使得用户可以更为灵活的提交偏移量。
##### 手动提交：
用户可以通过将 enable.auto.commit 设为 false，然后手动提交偏移量。基于用户需求手动提交偏移量可以分为两大类：
手动提交当前偏移量：即手动提交当前轮询的最大偏移量；
手动提交固定偏移量：即按照业务需求，提交某一个固定的偏移量。
而按照 Kafka API，手动提交偏移量又可以分为同步提交和异步提交。
**同步提交**：
通过调用 consumer.commitSync() 来进行同步提交，不传递任何参数时提交的是当前轮询的最大偏移量。
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.of(100, ChronoUnit.MILLIS));
    for (ConsumerRecord<String, String> record : records) {
        System.out.println(record);
    }
    // 同步提交
    consumer.commitSync();
}
```
如果某个提交失败，同步提交还会进行重试，这可以保证数据能够最大限度提交成功，但是同时也会降低程序的吞吐量。
**异步提交**
为了解决同步提交降低程序吞吐量的问题，又有了异步提交的方案。
异步提交可以提高程序的吞吐量，因为此时你可以尽管请求数据，而不用等待 Broker 的响应。代码样例如下：
```java
while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.of(100, ChronoUnit.MILLIS));
    // 异步提交并定义回调
    consumer.commitAsync(new OffsetCommitCallback() {
        @Override
        public void onComplete(Map<TopicPartition, OffsetAndMetadata> offsets, Exception exception) {
          if (exception != null) {
             offsets.forEach((x, y) -> System.out.printf("topic = %s,partition = %d, offset = %s \n", x.topic(), x.partition(), y.offset()));
            }
        }
    });
}
```
异步提交如果失败，错误信息和偏移量都会被记录下来。
尽管如此，异步提交存在的问题是，如果提交失败不能重试，因为重试可能会出现小偏移量覆盖大偏移量的问题。
虽然程序不能在失败时候进行自动重试，但是我们是可以手动进行重试。可以通过一个 Map<TopicPartition, Integer> offsets 来维护你提交的每个分区的偏移量，也就是异步提交的顺序，在每次提交偏移量之后或在回调里提交偏移量时递增序列号。然后当失败时候，你可以判断失败的偏移量是否小于你维护的同主题同分区的最后提交的偏移量，如果小于则代表你已经提交了更大的偏移量请求，此时不需要重试，否则就可以进行手动重试。
**同步和异步组合提交**：
当发生关闭消费者或者再均衡时，一定要确保能够提交成功，为了保证性能和可靠性，又有了同步和异步组合提交的方式。也就是在消费者关闭前组合使用commitAsync()方法和commitSync()方法。代码样例如下：
```java
try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.of(100, ChronoUnit.MILLIS));
        for (ConsumerRecord<String, String> record : records) {
            System.out.println(record);
        }
        // 异步提交
        consumer.commitAsync();
    }
} catch (Exception e) {
    e.printStackTrace();
} finally {
    try {
        // 因为即将要关闭消费者，所以要用同步提交保证提交成功
        consumer.commitSync();
    } finally {
        consumer.close();
    }
}
```
##### 提交特定的偏移量
上面的提交方式都是提交当前最大的偏移量，但如果需要提交的是特定的一个偏移量呢？
只需要在重载的提交方法中传入偏移量参数即可。代码样例如下：
```java
// 同步提交特定偏移量
commitSync(Map<TopicPartition, OffsetAndMetadata> offsets)
// 异步提交特定偏移量
commitAsync(Map<TopicPartition, OffsetAndMetadata> offsets, OffsetCommitCallback callback)
```
### 结束消费
上面的消费过程都是以无限循环的方式来演示的，那么如何来优雅地停止消费者的轮询呢。
Kafka 提供了 consumer.wakeup() 方法用于退出轮询。
如果确定要退出循环，需要通过另一个线程调用consumer.wakeup()方法；如果循环运行在主线程里，可以在ShutdownHook里调用该方法。
它通过抛出 WakeupException 异常来跳出循环。需要注意的是，在退出线程时最好显示的调用 consumer.close() , 此时消费者会提交任何还没有提交的东西，并向群组协调器发送消息，告知自己要离开群组，接下来就会触发再均衡 ，而不需要等待会话超时。
下面的示例代码为监听控制台输出，当输入 exit 时结束轮询，关闭消费者并退出程序：
```java
// 调用wakeup优雅的退出轮询
final Thread mainThread = Thread.currentThread();
new Thread(() -> {
    Scanner sc = new Scanner(System.in);
    while (sc.hasNext()) {
        if ("exit".equals(sc.next())) {
            consumer.wakeup();
            try {
                // 等待主线程完成提交偏移量、关闭消费者等操作
                mainThread.join();
                break;
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}).start();

try {
    while (true) {
        ConsumerRecords<String, String> records = consumer.poll(Duration.of(100, ChronoUnit.MILLIS));
        for (ConsumerRecord<String, String> rd : records) {
            System.out.printf("topic = %s,partition = %d, key = %s, value = %s, offset = %d,\n", rd.topic(), rd.partition(), rd.key(), rd.value(), rd.offset());
        }
    }
} catch (WakeupException e) {
    // 无需处理此异常
} finally {
    consumer.close();
}
```

### 关注我的公众号，获取更多关于面试、技术的文章及福利资源。

![Dali王的技术博客公众号](/pictures/Dali王的技术博客公众号.jpg)

**【参考资料】**
《Kafka 权威指南》
