---
title: 分布式系统ID生成方案汇总
categories:
    - '分布式'
tags:
    - 分布式ID
    - 数据库
    - Redis
    - snowflake
---


在分布式系统中，需要对大量的数据、消息、请求等进行唯一的标识，例如分布式数据库的`ID`需要满足唯一且多数据库同步，在单一系统中，使用数据库自增主键可以满足需求，但是在分布式系统中就需要一个能够生成全局唯一`ID`的系统，而且还要满足高可用。

<!--more-->

## 数据库自增长字段

本文只整理`MySQL`的自增字段方案，`Oracle`和`SQL Server`的自增长方案就不介绍了。

`MySQL`自增列使用`auto_increment`标识字段达到自增，在创建表时将某一列定义为`auto_increment`，则改列为自增列。这定了`auto_increment`的列必须建立索引。

### `auto_increment`使用说明

-  如果把一个NULL插入到一个auto_increment数据列中，MySQL将自动生成下一个序列编号。编号从1开始，并以1为基数递增；

-  把0插入auto_increment数据列的效果与插入NULL值一样，但是不建议这样做，还是以插入NULL值为好；
-  当插入记录时，没有为auto_increment明确指定值，则等同于插入NULL值；
-  当插入记录时，如果为auto_increment数据列明确指定了一个数值，则会出现两种情况，情况一，如果插入的值与已有的编号重复，则会出现出错信息，因为auto_increment数据列的值必须是唯一的；情况二，如果插入的值大于已编号的值，则会把该值插入到数据列中，并使在下一个编号将这个新值开始递增。也即可以跳过一些编号；
-  如果用update命令更新自增列，如果列值与已有的值重复，则会出错。如果大于已有值，则下一个编号从该值开始递增。


### 相关配置

`MySQL`中的自增长字段，在做数据库的主主同步时需要在参数文件中设置自增长的两个相关配置：

- `auto_increment`：自增长字段从哪个数开始，取值范围是：1~65535
- `auto_increment_increment`：自增长字段每次递增的量，即步长，默认值是1，取值范围是1~65535

优化方案：在配置集群的`MySQL`时，需要将n台服务器的`auto_increment_increment`都配置为`n`，而要把`auto_increment_offset`分别配置为`1，2，……，n`。这样才可以避免多台服务器更新时自增长字段的值之间出现冲突。

### 优缺点

#### 优点：

- 很小的数据存储空间，简单，代码方便，性能可以接受
- 数字`ID`天然排序，容易记忆，对分页或者需要排序的结果很有帮助

#### 缺点：

- 如果存在大量的数据，可能会超出自增长的取值范围
- 在单个数据库或读写分离或一主多从的情况下，只有一个主库可以生成，有单点故障的风险
- 很难处理分布式存储的数据表，尤其是需要合并表的情况下
- 安全性低，因为是有规律的，容易被非法获取数据



## UUID

`UUID`全称是`Universally Unique Identifier`，即通用唯一识别码。

`UUID`是由一组32位数的16进制数字所构成，所以理论上`UUID`的总数为`16^32=2^128`，约等于`3.4*10^38`。

`UUID`的标准形式包括32个16进制数字，以连字号分为5段，例如：`1FA504B0-2F82-21D3-0A0C-1205E72C3301`。`UUID`经由一定的算法机器生成，为了保证`UUID`的唯一性，规范定义了包括网卡`MAC`地址、时间戳、名字空间、随机数或伪随机数、时序等元素，以及从这些元素生成`UUID`的算法。

### 优缺点

#### 优点

- 本地生成`ID`，不需要进行远程调用，时延低，性能好

#### 缺点

- `UUID`过长，16字节共128位，通常以36长度的字符串标识，很多场景不适用，比如用`UUID`做数据库索引字段
- 没有排序，无法保证趋势递增
- 不可读



## UUID的变种

### UUID to Int64

为了解决`UUID`不可读的问题，可以使用`UUID to Int64`的方法。

```java
/// <summary>
/// 根据GUID获取唯一数字序列
/// </summary>
public static long GuidToInt64()
{
    byte[] bytes = Guid.NewGuid().ToByteArray();
    return BitConverter.ToInt64(bytes, 0);
}
```

### Comb（combined guid/timestamp）

为了解决`UUID`无序的问题，`NHibernate`在其主键生成方法中提供了`Comb`算法。保留`GUID`的10个字节，用另6个字节表示`GUID`生成的时间（`DateTime`）。

```java
/// <summary> 
/// Generate a new <see cref="Guid"/> using the comb algorithm. 
/// </summary> 
private Guid GenerateComb()
{
    byte[] guidArray = Guid.NewGuid().ToByteArray();
 
    DateTime baseDate = new DateTime(1900, 1, 1);
    DateTime now = DateTime.Now;
 
    // Get the days and milliseconds which will be used to build    
    //the byte string    
    TimeSpan days = new TimeSpan(now.Ticks - baseDate.Ticks);
    TimeSpan msecs = now.TimeOfDay;
 
    // Convert to a byte array        
    // Note that SQL Server is accurate to 1/300th of a    
    // millisecond so we divide by 3.333333    
    byte[] daysArray = BitConverter.GetBytes(days.Days);
    byte[] msecsArray = BitConverter.GetBytes((long)
      (msecs.TotalMilliseconds / 3.333333));
 
    // Reverse the bytes to match SQL Servers ordering    
    Array.Reverse(daysArray);
    Array.Reverse(msecsArray);
 
    // Copy the bytes into the guid    
    Array.Copy(daysArray, daysArray.Length - 2, guidArray,
      guidArray.Length - 6, 2);
    Array.Copy(msecsArray, msecsArray.Length - 4, guidArray,
      guidArray.Length - 4, 4);
 
    return new Guid(guidArray);
}
```



## Redis生成分布式ID

`Redis`是单线程的，并且提供了原子操作`INCR`和`INCRBY`，也可以用来生成高性能的分布式`ID`。

如果使用`Redis`集群来生成分布式`ID`的话，需要做同数据库`ID`相似的配置：起点与步长。

另外虽然`INCR`和`INCRBY`是原子性的，但是如果对获取`Id`进行了封装，那么要考虑对封装的方法进行线程安全性的考量。

### 优点

- 不依赖数据库，灵活方便，且性能优于数据库
- 数字`ID`天然排序，对分页或者需要排序的结果很有帮助

### 缺点

- 需要引入`Redis`
- 需要编码和配置的工作量比较大



## Twitter的snowflake算法

`snowflake`是`Twitter`的开源分布式`ID`生成算法，其生成结构如下(每部分用-分开):

```
0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000
```

第一位为未使用，接下来的41位为毫秒级时间(41位的长度可以使用69年)，然后是5位`datacenterId`和5位`workerId`(10位的长度最多支持部署1024个节点） ，最后12位是毫秒内的计数（12位的计数顺序号支持每个节点每毫秒产生4096个`ID`序号）

一共加起来刚好64位，为一个`Long`型。(转换成字符串后长度最多19)

`snowflake`生成的ID整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞（由`datacenter`和`workerId`作区分），并且效率较高。经测试`snowflake`每秒能够产生26万个`ID`。

官网：https://github.com/twitter-archive/snowflake

### Java版本的源码

```java
/**
 * Twitter_Snowflake<br>
 * SnowFlake的结构如下(每部分用-分开):<br>
 * 0 - 0000000000 0000000000 0000000000 0000000000 0 - 00000 - 00000 - 000000000000 <br>
 * 1位标识，由于long基本类型在Java中是带符号的，最高位是符号位，正数是0，负数是1，所以id一般是正数，最高位是0<br>
 * 41位时间截(毫秒级)，注意，41位时间截不是存储当前时间的时间截，而是存储时间截的差值（当前时间截 - 开始时间截)
 * 得到的值），这里的的开始时间截，一般是我们的id生成器开始使用的时间，由我们程序来指定的（如下下面程序IdWorker类的startTime属性）。41位的时间截，可以使用69年，年T = (1L << 41) / (1000L * 60 * 60 * 24 * 365) = 69<br>
 * 10位的数据机器位，可以部署在1024个节点，包括5位datacenterId和5位workerId<br>
 * 12位序列，毫秒内的计数，12位的计数顺序号支持每个节点每毫秒(同一机器，同一时间截)产生4096个ID序号<br>
 * 加起来刚好64位，为一个Long型。<br>
 * SnowFlake的优点是，整体上按照时间自增排序，并且整个分布式系统内不会产生ID碰撞(由数据中心ID和机器ID作区分)，并且效率较高，经测试，SnowFlake每秒能够产生26万ID左右。
 */
public class SnowflakeIdWorker {

    // ==============================Fields===========================================
    /** 开始时间截 (2015-01-01) */
    private final long twepoch = 1420041600000L;

    /** 机器id所占的位数 */
    private final long workerIdBits = 5L;

    /** 数据标识id所占的位数 */
    private final long datacenterIdBits = 5L;

    /** 支持的最大机器id，结果是31 (这个移位算法可以很快的计算出几位二进制数所能表示的最大十进制数) */
    private final long maxWorkerId = -1L ^ (-1L << workerIdBits);

    /** 支持的最大数据标识id，结果是31 */
    private final long maxDatacenterId = -1L ^ (-1L << datacenterIdBits);

    /** 序列在id中占的位数 */
    private final long sequenceBits = 12L;

    /** 机器ID向左移12位 */
    private final long workerIdShift = sequenceBits;

    /** 数据标识id向左移17位(12+5) */
    private final long datacenterIdShift = sequenceBits + workerIdBits;

    /** 时间截向左移22位(5+5+12) */
    private final long timestampLeftShift = sequenceBits + workerIdBits + datacenterIdBits;

    /** 生成序列的掩码，这里为4095 (0b111111111111=0xfff=4095) */
    private final long sequenceMask = -1L ^ (-1L << sequenceBits);

    /** 工作机器ID(0~31) */
    private long workerId;

    /** 数据中心ID(0~31) */
    private long datacenterId;

    /** 毫秒内序列(0~4095) */
    private long sequence = 0L;

    /** 上次生成ID的时间截 */
    private long lastTimestamp = -1L;

    //==============================Constructors=====================================
    /**
     * 构造函数
     * @param workerId 工作ID (0~31)
     * @param datacenterId 数据中心ID (0~31)
     */
    public SnowflakeIdWorker(long workerId, long datacenterId) {
        if (workerId > maxWorkerId || workerId < 0) {
            throw new IllegalArgumentException(String.format("worker Id can't be greater than %d or less than 0", maxWorkerId));
        }
        if (datacenterId > maxDatacenterId || datacenterId < 0) {
            throw new IllegalArgumentException(String.format("datacenter Id can't be greater than %d or less than 0", maxDatacenterId));
        }
        this.workerId = workerId;
        this.datacenterId = datacenterId;
    }

    // ==============================Methods==========================================
    /**
     * 获得下一个ID (该方法是线程安全的)
     * @return SnowflakeId
     */
    public synchronized long nextId() {
        long timestamp = timeGen();

        //如果当前时间小于上一次ID生成的时间戳，说明系统时钟回退过这个时候应当抛出异常
        if (timestamp < lastTimestamp) {
            throw new RuntimeException(
                    String.format("Clock moved backwards.  Refusing to generate id for %d milliseconds", lastTimestamp - timestamp));
        }

        //如果是同一时间生成的，则进行毫秒内序列
        if (lastTimestamp == timestamp) {
            sequence = (sequence + 1) & sequenceMask;
            //毫秒内序列溢出
            if (sequence == 0) {
                //阻塞到下一个毫秒,获得新的时间戳
                timestamp = tilNextMillis(lastTimestamp);
            }
        }
        //时间戳改变，毫秒内序列重置
        else {
            sequence = 0L;
        }

        //上次生成ID的时间截
        lastTimestamp = timestamp;

        //移位并通过或运算拼到一起组成64位的ID
        return ((timestamp - twepoch) << timestampLeftShift) //
                | (datacenterId << datacenterIdShift) //
                | (workerId << workerIdShift) //
                | sequence;
    }

    /**
     * 阻塞到下一个毫秒，直到获得新的时间戳
     * @param lastTimestamp 上次生成ID的时间截
     * @return 当前时间戳
     */
    protected long tilNextMillis(long lastTimestamp) {
        long timestamp = timeGen();
        while (timestamp <= lastTimestamp) {
            timestamp = timeGen();
        }
        return timestamp;
    }

    /**
     * 返回以毫秒为单位的当前时间
     * @return 当前时间(毫秒)
     */
    protected long timeGen() {
        return System.currentTimeMillis();
    }

    //==============================Test=============================================
    /** 测试 */
    public static void main(String[] args) {
        SnowflakeIdWorker idWorker = new SnowflakeIdWorker(0, 0);
        for (int i = 0; i < 1000; i++) {
            long id = idWorker.nextId();
            System.out.println(Long.toBinaryString(id));
            System.out.println(id);
        }
    }
}
```

### 优点

- 不依赖数据库，灵活方便，且性能优于数据库
- `Id`按照时间在单机上是递增的

### 缺点

- 在单机上是递增的，但是在分布式环境中，每台机器上的时钟不可能完全同步，可能会出现不是全局递增的情况



原文地址：[https://wangjinlong.xyz/2018/09/17/distributed-system-id-generation/](https://wangjinlong.xyz/2018/09/17/distributed-system-id-generation/)

