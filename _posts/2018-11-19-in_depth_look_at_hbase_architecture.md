---
layout: article
title: 【译】深入了解HBase体系结构
tags: HBase
---

> HBase系列博客的翻译，本文深入介绍了HBase体系结构及其对于NoSQL数据存储解决方案的主要优点
>
> 原文链接：[An In-Depth Look at the HBase Architecture](https://mapr.com/blog/in-depth-look-hbase-architecture/)

<!--more-->

## HBase架构组件

从物理存储上来看，HBase基于主从架构，由三种类型的服务组成。RegionServer提供读写数据的服务。当访问数据时，客户端直接与HBase的RegionServer进行通信。Region的分配和DDL操作(创建、删除表)由HBase的Master进程来处理。Zookeeper作为HDFS的一部分，负责维护集群状态。

Hadoop的DataNode用来储存RegionServer管理的数据。所有的HBase数据都存储在HDFS文件中。RegionServer与HDFS的DataNode是并列的关系，以保证RegionServer所管理的数据具有数据局部性(data locality, 把数据放在需求的地方)。当HBase的数据被写入时是本地的，但是当一个Region被移动时，在被压缩之前数据不是本地的。 

NameNode维护包含文件的所有物理数据块的元数据信息。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig1.png)

## Regions

HBase表按照行键的范围被水平划分到多个region中。一个region中包含了表中从region的开始键到结束键的所有行。Region被分配给集群中的节点，称为“RegionServer”，这些节点为提供读写数据服务。一个RegionServer大约可以服务1000个region。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig2.png)



## HBase HMaster

HBase Master负责处理region的分配和DDL操作(创建、删除表)。

另外Master还负责：

- 协调RegionServer
  - 在启动时分配region，或者在恢复和负载均衡时重新分配region
  - 监视集群中的所有RegionServer实例(监听来自zookeeper的通知) 
- 管理功能
  - 创建、删除和更新表的接口

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig3.png)

## 协调器ZooKeeper

HBase使用ZooKeeper作为分布式协调服务来维护集群中的服务器状态。ZooKeeper维护哪些服务器是可用的，并提供服务器故障通知。ZooKeeper使用信息一致性来保证共同的共享状态。注意为了达成一致性，至少应该有3到5台机器。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig4.png)



## 组件如何协同工作

ZooKeeper用来协调分布式系统成员的共享状态信息。RegionServers和HMaster与ZooKeeper之间建立会话连接。ZooKeeper通过心跳信号维护与临时节点(ephemeral node)的会话连接。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig5.png)

每个RegionServer会创建一个临时节点。HMaster监视这些节点以发现可用的RegionServer和故障的服务器。HMasters之间竞争创建临时节点，ZooKeeper确定第一个节点并将其作为master，确保只有一个master节点是激活的。激活的HMaster开始向ZooKeeper发送心跳信号，而未被激活的HMaster监听激活的HMaster的故障通知。

如果RegionServer或HMaster未能成功发送心跳信号，则会话过期并删除相应的临时节点。删除节点的通知消息发送会用于更新的监听器，激活的HMaster会恢复故障的RegionServer。如果激活的HMaster故障，未激活的HMaster会取而代之。

## HBase初次读写

HBase的Catalog表中有一张特殊的表称为META表，这张表包含了集群中region的位置。META表的位置由ZooKeeper储存。

下面是客户端第一次读写HBase时会发生的情况：

1. 客户端从ZooKeeper获取拥有META表的RegionServer。
2. 客户端通过查询.META.服务器来获取与想要访问的行键对应的RegionServer。客户端会将这些信息连同META表的位置一起做缓存。
3. 从相应的RegionServer中获取目标行。

对于以后的读取操作，客户端使用缓存来检索META表位置和之前读取的行键。长此以往，客户端不需要再查询META表，除非由于region的移动而出现遗漏，才会重新查询并更新缓存。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig6.png)

## HBase Meta Table

- META表是一个HBase表，保存了系统中所有region的列表。
- .META.表类似于B树
- .META.表的结构如下：
  - Key: region start key, region id
  - Values: RegionServer

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig7.png)

## RegionServer组件

RegionServer运行在HDFS数据节点上，有以下组件：

- WAL：WAL(Write Ahead Log)是分布式文件系统中的一个文件，用来存储尚未被持久化的新数据，以便于在出现故障时进行数据恢复。
- BlockCache：读缓存，将频繁读取的数据存储在内存中。最近使用最少的数据在缓存满是会被删除。
- MemStore：写缓存，存储尚未被写入磁盘的新数据，在写入磁盘之前对数据进行排序。每个region中的每个列族有一个MemStore。
- HFile：将按照键值排序的行存储在磁盘上。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig8.png)

## HBase写步骤

当客户端发出Put请求时，第一步是将数据写入write-ahead log(即WAL)：

- 新的数据会被附加到存储在磁盘上的WAL文件的末尾。
- WAL用来恢复尚未持久化的数据，以防止服务器崩溃。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig9.png)

数据一旦写入WAL，就会被放置到MemStore中。然后Put请求的确认信息会返回给客户端。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig10.png)

## HBase MemStore

MemStore将按键值排序的更新数据存储在内存中，与将被存到HFile中的数据格式一样。每个列族有一个MemStore，并按顺序进行更新。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig11.png)

## HBase Region Flush

当MemStore中积累了足够多的数据时，会将整个数据集写入到HDFS中的新HFile中。HBase在每个列族中使用多个HFile，其中包含了实际cell或KeyValue实例。当MemStore中的有序键值数据以文件的形式刷新到磁盘上时，这些HFile也随之被创建。

注意这也是为什么HBase中列族数量是有限制的原因。每个列族有一个MemStore，数据满的时候会刷新到磁盘.HFile中存储了最后写入的序列号，以便让系统知道数据持久化的进度。

最高序列号存储在每个HFile中的元字段中，以反映持久化已结束的位置和要继续的位置。Region启动的时候，会读取序列号，并将最高值用于新的操作中。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig12.png)

## HBase HFile

数据存储在一个包含有序键/值的HFile中。当MemStore中积累了足够的数据时，整个有序键值集合会被写入到HDFS中的新HFile中。这是一个顺序写操作，因为避免了移动磁盘驱动器头，所以非常快速。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig13.png)

## HBase HFile结构

HFile包含一个多层索引，以方便HBase检索数据，而不需要去读取整个文件。多级索引类似于B+树：

- 键值对按递增顺序存储
- 索引按行键指向64KB大小的“块”中的键值数据
- 每个块都有自己的叶索引
- 每个块的最后一个键放在中间索引中
- 根索引指向中间索引

Trailer指向元数据块，并被写入到文件中持久化数据的末尾。Trailer还包含了布鲁姆过滤器和时间范围信息。布鲁姆过滤器可以协助过滤不包含特定行键的文件。如果文件不在读取的时间范围内，可以基于时间范围信息跳过该文件。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig14.png)

## HFile索引

上文中讨论的索引，是HFile打开并保存在内存中的时候被加载。这样就可以使用单个磁盘寻道执行查找。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig15.png)

## HBase读合并

通过上面的介绍可以看到，一行对应的键值cell可能位于多个不同的位置上。已持久化的cell位于HFile中，最近更新的cell在MemStore中，最近读取的cell在BlockCache中。那么当读取一行内容的时候，系统通过以下步骤中合并来自BlockCache、MemStore和HFile中的键值：

1.  首先，扫描程序在BlockCache中寻找行cell。
2. 接下来再在MemStore中查找。
3. 如果扫描程序在MemStore和BlockCache中没有找到全部的行cell，HBase会使用BlockCache缓存和布鲁姆过滤器将HFile加载到内存，继续寻找其余的目标行cell。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig16.png)

如前面所讲到的，每个MemStore可能会对应多个HFile，这意味着对于读取，可能需要检查多个文件，进而会影响性能。这种现象称为读取放大(read amplification)。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig17.png)

## HBase Minor Compaction

为了避免读取放大的情况，HBase会自动选择一些较小的HFile，将它们重写为若干较大的HFile。这一过程称为轻度压实(minor compaction)。轻度压实通过合并排序，将较小的文件重写成少量大文件，以减少存储文件的数量。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig18.png)

## HBase Major Compaction

高度压实(Major compaction)将region中所有的HFile进行合并和重写，为每个列族生成一个HFile。在这个过程中，有删除标记或过期的cell会被移除。高度压实可以提供读取性能，但是由于需要重写所有的文件，这个过程会产生大量的磁盘IO和网络流量，这种现象称为写放大(write amplification)。

高度压实可以设置按计划自动运行，但由于写放大，通常安排在周末或者晚上进行。由于服务器故障或者负载均衡，高度压实会使得原先的远程数据文件变为RegionServer的本地文件。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig19.png)

## Region = Contiguous Keys

在继续下文之前，先对region的概念做一个快速回顾：

- 一张表可以被水平地划分为一个或多个region。Region中包含了从开始键到结束键范围内的连续有序的行
- 每个region默认大小为1GB
- 表的region通过RegionServer为客户端提供服务
- 一个RegionServer可以服务大约1000个region(可能属于一个或多个不同的表)

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig20.png)

## Region Split

一开始每个表有一个region，当region变得过大时，会分裂为两个子region。两个子region(各自代表原始region的一半)在同一个RegionServer上并行打开，然后报告给HMaster。出于负载均衡的原因，HMaster可能会将新region转移到其他服务器。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig21.png)

## 读负载平衡

上文提到，region的分割最初发生在同一RegionServer上，但是出于负载均衡的原因，HMaster可能会将新region转移到其他服务器上。这导致RegionServer需要为远程的HDFS节点数据提供服务，直到高度压实操作将数据文件移动到RegionServer的本地节点。HBase数据在一开始写入时是本地的，但是当一个region被移动(由于负载均衡或数据恢复)时，在高度压实之前它都不会存储在本地。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig22.png)

## HDFS数据复制

所有的读写操作都要经过主节点。HDFS会复制WAL和HFile块，其中HFile块的复制是自动的。HBase在存储文件是依赖于HDFS提供数据安全性。当数据写入到HDFS时，一个副本在本地写入，然后复制到一个二级节点，第三个副本写入到一个三级节点。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig23.png)

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig24.png)

## HBase崩溃恢复

当RegionServer故障时，在检测和恢复步骤之前，崩溃的region是不可用的。在失去RegionServer的心跳信号时，ZooKeeper会判定节点故障，然后通知HMaster来处理。

当HMaster获知某个RegionServer已崩溃时，会将崩溃的RegionServer上的region重新分配到可用的RegionServer上。为了恢复崩溃的RegionServer中没有刷新到磁盘上的MemStore操作，HMaster将原RegionServer上的WAL文件分割为多个文件存储到新的RegionServer的数据节点上。然后，每个新RegionServer重新执行对应的WAL文件，重新构建MemStore。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig25.png)

## 数据恢复

WAL文件包含一个编辑列表，其中一个编辑表示一个put或delete操作。编辑是按时间顺序写入的，因此，为了持久化，会附加到存储在磁盘上的WAL文件的末尾。

如果数据仍然在内存中且没有持久化到HFile中的时候出现故障，会发生什么情况？通过读取WAL文件重新执行WAL，把其中包含的编辑添加到当前的MemStore中并对其进行排序。最后，MemStore将所做的更改刷新到HFile中。

![avatar](https://mapr.com/blog/in-depth-look-hbase-architecture/assets/blogimages/HBaseArchitecture-Blog-Fig26.png)

## Apache HBase架构的优点

- 强一致性模型
  - 当写入返回时，所有的读取得到的都是同样的值
- 弹性伸缩
  - 当数据过大时分割region
  - 使用HDFS扩展和复制数据
- 内置数据恢复
  - 使用WAL(Write Ahead Log)，类似于文件系统中的记录日志
- 与Hadoop集成
  - 在HBase上使用MapReduce是非常简单的

## Apache HBase的缺陷

- 业务上的连续性与可靠性
  - WAL的重新执行操作缓慢
  - 缓慢复杂的崩溃恢复过程
  - 高度压实操作的IO压力

