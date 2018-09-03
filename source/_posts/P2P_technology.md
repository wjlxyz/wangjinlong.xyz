---
title: P2P技术揭秘（一）：P2P技术概述
categories:
    - '技术'
    - '网络'
    - 'P2P技术'
tags:
    - 网络
    - P2P技术
---


为了更好的理解区块链的底层设计，我觉得有必要完整的学习一下P2P网络的技术，为此专门找了《P2P技术揭秘——P2P网络技术原理与典型系统开发》这本书来读。这篇博客就是记录学习过程中的一些笔记。

<!--more-->

## P2P技术概述

### `P2P`在数学上的严格定义

- 有某`网络N`，是架构在`Internet`上的网络，满足`Internet`网络的所有基本特征；
- 在`N网络`中，存在两种基本的行为模式，一种行为定义为`P`，是生产资源（提供资源的行为），另一种行为定义为`C`，是消费资源（接受资源）的行为；
- 组成`N网络`的所有网络节点（`Peer`）之间是对等的关系，且同时具备行为`P`和行为`C`；
- 在`N网络`中，各`Peer`之间以无中介的、对等的方式进行双向交换，以执行`P`和`C`的功能；
- `N网络`依赖`Peer`的存在而存在，且`Peer`可以自由地加入或退出；
- 当有`Peer`加入或退出时，`N`仍保持组织、结构等特性不发生改变。

满足以上6个条件的网络N，就可以说是P2P网络。在这个网络汇总，实现了资源的生产和消费的对等平衡，实现了信息和服务在一个个人或对等设备与另一个个人与对等设备间的双向流动。一个完整的P2P网络模型如图所示：

![P2P_network_structure](/pictures/p2p_network_secret/P2P_network_structure.png)

### `P2P`发展实例

1. `P2P`文件共享系统——`eMule`（基于`eDonkey`的网络协议）

2. `P2P`下载技术——迅雷

   迅雷的技术主要分成两个部分：

   - 一部分是对现有`Internet`下载资源的搜索和整合，将现有`Internet`上的下载资源进行校验，将相同检验值的统一资源定位（`URL`）信息进行聚合。当用户单击某个下载链接时，迅雷服务器按照一定的策略返回该URL信息所在聚合的子集，并将该用户的信息返回给迅雷服务器。
   - 另一部分是迅雷客户端通过多资源多线程下载所需要的文件，提高下载速率。

3. `P2P`文件传输技术——`BT`

   `BT`协议是一个网络文件传输协议，它能够实现点对点文件分享的功能。

   `BT`中，`BT`种子文件信息是根据对目标文件的计算生成的，计算结果根据`BT`协议内的B编码规则进行编码。它的主要原理是需要把提供下载的源文件虚拟分成大小相等的块，块大小必须为`2k`的整数次方，并把每个块的索引信息和`Hash`验证码写入`.torrent`文件中，所以`.torrent`文件简单地说就是被下载的源文件的“索引”信息。

4. `P2P`视频分发技术——`PPLive`

   `PPLive`是一款用于互联网上大规模视频直播的免费软件，是基于`P2P`技术的流媒体发布和传输方面的典型代表，也是`P2P`技术在视频直播、点播应用上的成功范例。

5. `P2P`网络语音通信技术——`Skype`

### `P2P`的技术特点及存在的问题

1. `P2P`的网络虚拟性
2. `P2P`的拓扑动态性
3. `P2P`的网络健壮性
4. `P2P`的负载均衡
5. `P2P`在应用中的高性价比
6. `P2P`是处于应用层的网络

### `P2P`发展带来的问题

1. 知识产权问题
2. 黑客和病毒入侵问题
3. 敏感信息的泄漏问题
4. 带宽问题



## `P2P`网络拓扑结构

根据拓扑结构的关系可以将`P2P`网络拓扑结构分为4种形式：

- 集中式拓扑（也称中心化拓扑）
- 全分布式结构化拓扑（也称DHT网络）
- 全分布式非结构化拓扑
- 混合式拓扑

### 集中式的`P2P`网络拓扑

集中式`P2P`网络模型结构图如下：

![centeralized_topology](/pictures/p2p_network_secret/centeralized_topology.png)

### 全分布式结构化`P2P`网络拓扑

全分布式结构化拓扑的`P2P`网络主要是采用分布式散列表技术来组织网络中的节点。分布式散列表的全称是`Distributed Hash Table`，简写为`DHT`。

结构化`P2P`网络中，每个节点都有固定的地址，整个网络具有相对稳定而规则的拓扑结构。依赖拓扑结构可以给网络的每个节点制定一个逻辑地址，并把地址和节点的位置对应起来。

### 全分布式非结构化的`P2P`网络

这种结构的覆盖网络一般采用基于完全随机图的组织方式，节点度数服从`Power-law`规律（幂次法则），从而能够较快发现目的节点。

非结构化网络中，节点没有制定的逻辑地址，采用随机算法或者启发策略加入网络，网络拓扑随着节点的变迁和网络通信的进行而发生演变。

### 混合式`P2P`网络结构

混合式`P2P`网络模型结构图如下：

![partially_decentralized_topology](/pictures/p2p_network_secret/partially_decentralized_topology.png)



## `P2P`网络的搜索技术

### 什么是`P2P`搜索

`P2P`网络的根本思想就在于对等和共享。在`P2P`系统中，资源是分散在各个节点之上的，并且节点频繁地加入或退出都相当自由，几乎没有规律可循。这些都使得`P2P`系统及整个`P2P`系统的资源都处于不断的变化之中，那么如何找到并定位这些节点和资源，就是`P2P`需要解决的一个重要问题。

所谓`P2P`搜索技术，就是一种`P2P`资源的发现和定位技术，通过搜索算法来发现、查找`P2P`网络中，在时间和空间上都处于动态的变化的节点信息和资源存储信息，以最大限度、最快速度、尽可能多且准确的发现节点上的资源。

相比于`Web`搜索，`Web`搜索是面向全网的，凡是加入到`Internet`中的主机节点理论上都适用于`Web`搜索，`P2P`搜索则是面向`P2P`网络架构的。

### `P2P`网络搜索技术

1. 集中式`P2P`网络采用目录索引机制。
2. 结构化`P2P`网络
   1. 基于`DHT`的搜索实现
   2. `Chord`网络搜索技术
   3. `Pastry`网络搜索技术
   4. `CAN`与`Tapestry`
3. 非结构化`P2P`网络的搜索方法（盲目搜索和启发式搜索）
   1. `Flooding`搜索算法
   2. 迭代递增搜索
   3. 启发式泛洪搜索
   4. `Random Walk`搜索方法
