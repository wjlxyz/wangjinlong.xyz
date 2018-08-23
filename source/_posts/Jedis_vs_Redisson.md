---
title: Jedis与Redisson选型对比
---

### 1　　概述

#### 1.1       主要内容

本文的主要内容为对比Redis的两个框架：`Jedis`与`Redisson`，分析各自的优势与缺点，为项目中`Java`缓存方案中的Redis编程模型的选择提供参考。

### 2   `Jedis`与`Redisson`对比

#### 2.1    概况对比

`Jedis`是`Redis`的`Java`实现的客户端，其`API`提供了比较全面的`Redis`命令的支持；`Redisson`实现了分布式和可扩展的`Java`数据结构，和`Jedis`相比，功能较为简单，不支持字符串操作，不支持排序、事务、管道、分区等`Redis`特性。`Redisson`的宗旨是促进使用者对`Redis`的关注分离，从而让使用者能够将精力更集中地放在处理业务逻辑上。

#### 2.2   编程模型

`Jedis`中的方法调用是比较底层的暴露的`Redis`的`API`，也即`Jedis`中的`Java`方法基本和`Redis`的`API`保持着一致，了解`Redis`的`API`，也就能熟练的使用`Jedis`。而`Redisson`中的方法则是进行比较高的抽象，每个方法调用可能进行了一个或多个`Redis`方法调用。

如下分别为`Jedis`和`Redisson`操作的简单示例：

`Jedis`设置`key-value`与`set`操作：

   ```java
   Jedis jedis = …;
   jedis.set("key", "value");
   List<String> values = jedis.mget("key", "key2", "key3");
   // Redisson操作map：
   Redisson redisson = …
   RMap map = redisson.getMap("my-map"); // implement java.util.Map
   map.put("key", "value");
   map.containsKey("key");
   map.get("key");
   ```

#### 2.3    可伸缩性

`Jedis`使用阻塞的`I/O`，且其方法调用都是同步的，程序流需要等到`sockets`处理完`I/O`才能执行，不支持异步。`Jedis`客户端实例不是线程安全的，所以需要通过连接池来使用`Jedis`。

`Redisson`使用非阻塞的`I/O`和基于`Netty`框架的事件驱动的通信层，其方法调用是异步的。`Redisson`的`API`是线程安全的，所以可以操作单个`Redisson`连接来完成各种操作。

#### 2.4    数据结构

`Jedis`仅支持基本的数据类型如：`String`、`Hash`、`List`、`Set`、`Sorted Set`。

`Redisson`不仅提供了一系列的分布式`Java`常用对象，基本可以与`Java`的基本数据结构通用，还提供了许多分布式服务，其中包括（`BitSet`, `Set`, `Multimap`, `SortedSet`, `Map`, `List`, `Queue`, `BlockingQueue`, `Deque`, `BlockingDeque`, `Semaphore`, `Lock`, `AtomicLong`,`CountDownLatch`, `Publish / Subscribe`, `Bloom filter`, `Remote service`, `Spring cache`, `Executor service`, `Live Object service`, `Scheduler service`）。

在分布式开发中，`Redisson`可提供更便捷的方法。

#### 2.5    第三方框架整合

1. `Redisson`提供了和`Spring`框架的各项特性类似的，以`Spring XML`的命名空间的方式配置`RedissonClient`实例和它所支持的所有对象和服务；
2. `Redisson`完整的实现了`Spring`框架里的缓存机制；
3.  `Redisson`在`Redis`的基础上实现了`Java`缓存标准规范；
4.   `Redisson`为`Apache Tomcat`集群提供了基于Redis的非黏性会话管理功能。该功能支持`Apache Tomcat`的6、7和8版。
5. `Redisson`还提供了`Spring Session`会话管理器的实现。