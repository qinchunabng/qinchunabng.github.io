---
layout: post
title: Redis集群
categories: [Redis]
description: Redis集群
keywords: Redis, 集群
---

### Redis集群中客户端和服务端的角色

在redis集群中，集群节点除了存储数据，还需要维护集群的状态信息，映射key到对应的节点。同时集群节点能够自动发现其他节点，检测到无法工作的节点，并且在master节点down掉的时候能够提升replica节点为master节点。

集群中的所有节点都需要通过TCP连接起来，使用一个二进制通信协议进行通信，这种方式叫做Redis集群总线。集群中每个节点都需要通过集群总线与集群中的其他节点进行连接。节点使用gossip协议传播集群信息，用于发现新的节点，通过发送ping消息来确定其他集群节点是否正常，并且发送特定信号所需的集群消息。集群总线也会用来在集群中传播订阅/发布的消息，并且可以用来手动触发节点下线的操作（手动触发节点下线指的是不是被redis集群检测出现的节点下线，而是由管理员手动触发的）。

由于集群节点不能代理请求，所以客户端的请求会被-MOVED和-ASK重定向到其他节点。客户端理论上可以发送请求到所有的节点，请求会被转发到正确的节点，所以客户端不需要存储集群的状态信息。但是客户端可以缓存key和节点的映射关系来提升性能。

### 写安全

Redis集群节点间数据复制是异步的，最后一次故障转移获得隐式合并的功能。这意味最新选举的主节点会替代其他的副本节点。同时在出现网络分区的时候，会存在一个写丢失的时间窗口。对于这个时间窗口大小，连接大部分主节点的客户端和连接小部分主节点的客户端区别较大。

连接到大部分主节点的客户端，Redis集群会尽可能的保证写不会丢失。下面一些场景可能会导致连接大部分主节点的写丢失：

1. 一个写操作发送到了master节点，并且master节点给客户端返回了响应，但是这个写操作可能没有同步到replica，因为主从节点之间数据复制是异步执行的。如果这个时候master节点挂了，从节点晋升为主节点，这个写入操作就丢失了。这种情况发生的概率比较小，虽然还是又可能发生的。
2. 另外一种理论上可能发生写丢失的场景如下：
   - 一个master节点因为网络分区变为网络不可达。
   - 然后一个replica节点被晋升为主节点。
   - 过了一段时间旧的master节点又变为网络可达。
   - 一些没有更新路由表的客户端在旧的master节点变为replica节点之前，执行了一些写入操作。

第二种情况不太可能发生，因为不能与其他大部分主节点通信的主节点，且时间没有达到故障转移的时间，不会接受新的写请求，并且当网络分区被修复，写操作在短时间内也是会被拒绝的，因为需要通知其他节点集群配置信息的更新。同时这种情况也需要客户端的路由表信息没有更新。

写操作落在小部分分区写丢失会有更大的时间窗口。例如，如果有客户端连接到分区中小部分分区，写丢失会丢失的比较多，因为发送到分区中小部分的master的写会因为被分区中大部分master节点故障转移而丢失。

一个master节点被故障转移，肯定是由于大部分的master节点对这个master节点网络不可达的时间至少超过NODE_TIMEOUT，所以如果网络分区在这个时间前被修复，就不会有写会丢失。如果网络分区超过NODE_TIMEOUT，所有分区小部分master节点写操作在这个时间之前都会丢失。但是当网络分区超过NODE_TIMEOUT，网络分区的小部分master节点会开始拒绝写操作，所以在小部分master节点变为不可用时有个最大的写丢失的时间窗口。但是，在这个时间点之后，小部分master节点的分区不会再接受写操作，就不会有写丢失。

### 可用性

在网络分区中的有小部分节点的分区中Redis集群是不可用的。假设网络分区中的有较多的节点那边,存在大部分master节点，且每个故障不可达的master节点都有一个replica节点，在过了NODE_TIMEOUT加上replica节点选举为master节点的时间（通常会花费1到2秒），集群会再次变得可用。

这意味着Redis集群在集群中小部分节点挂了，集群仍然可用。但是如果集群出现了大的网络分区，集群就会变为不可用。

假如集群中有N个master节点，并且没有master节点有一个replica节点，当有一个节点出现问题，集群中大多数侧仍然可用，当集群中有2个阶段出现问，集群仍然可用的概率是1-1/(N\*2-1)(因为第一个节点挂掉之后，集群中还剩下2N-1个节点，而这个没有replica的master节点出现问题的概率是1/(N\*2-1))。

举个例子，一个集群有5个master节点，每个master节点有1个replica节点，在两个节点出现问题之后，集群有1/(5*2-1)=11.11%的概率变为不可用。

可以通过replica迁移来避免以上问题，重新为没有replica节点的master节点配置replica节点。

### 性能

Redis集群不会代理请求到正确的节点，它会将客户端的请求重定向到正确的集群节点。最终客户端会获取到最新的key的集群节点映射关系，所以普通的请求是直接发送的相应的节点的。

因为数据复制是异步的，节点不会等待replica节点数据复制完成（除非使用WAIT命令）。

另外，因为多个key的命令只能在临近key之间执行，数据是不会在节点之间移动，除非执行了resharding。

一般的操作都是被单个节点处理的，所以一个拥有N个master节点的Redis集群是单节点Redis的性能的N倍。查询操作也是只需要一个RT，因为客户端会保留与服务端的长连接，所以集群的延迟与单节点也是一样的。