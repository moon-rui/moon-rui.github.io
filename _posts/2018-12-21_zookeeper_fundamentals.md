---
layout: article
title: Zookeeper基础
key: 2018-12-21_zookeeper_fundamentals
tags: Zookeeper
---

> 翻译自tutorialspoint上的zookeeper教程，介绍zookeeper中的基本概念。
>
> 原文链接：[Zookeeper - Fundamentals](https://www.tutorialspoint.com/zookeeper/zookeeper_fundamentals.htm)

<!--more-->

## Zookeeper架构
下图是Zookeeper的主从架构
![avatar](https://www.tutorialspoint.com/zookeeper/images/architecture_of_zookeeper.jpg)

下表是每个组件的说明

组件 | 说明
---|---
Client | 分布式应用程序集群中的一个节点，从Server端获取信息。在特定的时间间隔内，每个client向server发送消息以让server知道client是处于活跃状态的。类似的，server在client连接时发送确认信息。如果连接的server没有响应，client自动将消息重定向到其他server。
Server | Zookeeper Ensemble中的一个节点，为client提供所有服务。给client发送确认消息以告知server处于活跃状态。
Ensemble | 一组Zookeeper Server。构成一个Ensemble所需要的最少节点数为3.
Leader | 在有连接节点失败时执行自动恢复的Server节点。Leader在服务启动时被选出。
Follower | 遵循Leader指令的Server节点

## Hierarchical Namespace 分层命名空间
下图描述了用于内存表示的ZooKeeper文件系统的树形结构。ZooKeeper节点称为znode。每个znode由一个名称标识，并由一系列路径(/)分隔。
- 在图中，首先有一个以“/”分隔的根znode。在根节点下，有两个逻辑命名空间config和workers。
- config命名空间用于集中管理配置，workers命名空间用于命名。
- 在config命名空间下，每个znode最多可以存储1MB的数据。这与UNIX文件系统类似，只是父znode也可以存储数据。该结构的主要目的是存储同步数据并描述znode的元数据。这种结构称为ZooKeeper数据模型。
![avatar](https://www.tutorialspoint.com/zookeeper/images/hierarchical_namespace.jpg)

ZooKeeper数据模型中的每个znode都维护一个stat结构。stat只提供znode的元数据，由版本号、操作控制列表(Action control list, ACL)、时间戳和数据长度组成。
- 版本号：每个znode都有一个版本号，每当与znode关联的数据发生变化时，它对应的版本号也会增加。当多个zookeeper client试图在同一个znode上执行操作时，版本号的作用就非常重要了。
- Action Control List (ACL)：ACL是访问znode的身份验证机制。它管理所有的znode读写操作。
- 时间戳：时间戳表示znode从创建到修改的累积时间，通常以毫秒表示。Zookeeper从“Transaction ID”(zxid)识别znode的每次更改。Zxid是唯一的，它为每个事务维护时间，可以轻松地标识从一个请求到另一个请求所花费的时间。
- 数据长度：znode中存储的数据总量就是数据长度。最多可以存储1MB的数据。

### Znode的类型
Znode的类型包括持久znode(persistence)，序列znode(sequential)，临时znode(ephemeral)。
- Persistence znode：即使在创建对应znode的client断开连接后，Persistence znode仍然保持活跃状态。默认情况下，除非另外指定，否则所有znode都是持久的。
- Ephemeral znode：当client从ZooKeeper Ensemble中断开连接时，就会自动删除Ephemeral znode。由于这个原因，只有Ephemeral znode不允许进一步拥有子节点。Ephemeral znode被删除后，下一个合适的节点将填充它的位置。Ephemeral znode在Leader的选举中发挥重要作用。
- Sequential znode：Sequential znode可以是持久的，也可以是临时的。当一个新的znode被创建为一个Sequential znode时，ZooKeeper会通过将一个10位数的序列号附加到原始名称上来设置znode的路径。例如，如果将带有路径 /myapp的znode创建为Sequential znode, ZooKeeper会将路径设置为/myapp0000000001，并将下一个序列号设置为0000000002。如果同时创建了两个Sequential znode，ZooKeeper不会为每个znode使用相同的序列号。myapp的znode创建为Sequential znode在锁和同步机制中起着重要的作用。

## Sessions
Sessions对于Zookeeper的操作非常重要。session中的请求按照FIFO的顺序执行。当client连接到server，session会被建立，session id会分配给client。

client在特定时间间隔发送心跳信息以保持session有效。如果ZooKeeper Ensemble在服务开始时指定的时间段以上(会话超时)都没有接收到来自client的心跳信息，那么认定client失效。

session超时通常以毫秒表示。当session终止时，会话期间创建的ephemeral znode也会被删除。

## Watches
对于client来说，Watches是一种用于获取ZooKeeper Ensemble中的更改通知的简单机制。client可以在读取znode时设置watches。对于任何znode的更改，watches将发送通知给注册的client。

Znode更改是指与Znode关联的数据的修改或Znode子节点中的更改。Watches只会触发一次。如果client需要再次被通知，则必须通过另一个读操作来完成。当连接会话过期时，客户端将与服务器断开连接，相关的watches也将被删除。