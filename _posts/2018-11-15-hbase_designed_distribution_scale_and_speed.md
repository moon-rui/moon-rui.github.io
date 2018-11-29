---
layout: article
title: 【译】为分布式、大规模、低延迟而设计的HBase
key: 2018-11-15-hbase_designed_distribution_scale_and_speed
tags: HBase
---

> 继续HBase系列博客的翻译，本文是对HBase的概述，适合HBase的入门
>
> 原文链接：[HBase and MapR-DB: Designed for Distribution, Scale, and Speed](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/)

<!--more-->

Apache HBase是在Hadoop集群上运行的数据库。和传统关系型数据库不同的是，为了获得更大的可扩展性，HBase没有严格遵循传统系统的ACID(原子性、一致性、隔离性和持久性)特性。 存储在HBase中的数据也不需要像在RDBMS中那样严格契合表结构模式，这使得它非常适合存储非结构化或半结构化数据。

基于HBase可以构建大数据应用程序，但与使用传统关系型数据库进行开发相比，它提供了一些实现应用程序的不同方法。因此在本文中，将对HBase做简要的概述，介绍关系型数据库的局限性，并深入研究HBase数据模型的细节。

## 关系型数据库 vs. HBase - 数据存储模型 

首先，在讨论关系型数据库的局限性之前，让我们先看看关系型数据库的优点：

- 关系型数据库提供了一个标准的持久化模型
- SQL已经成为数据操作的实际标准模型
- 关系型数据库能够有效管理事务的并发性
- 关系型数据库有很多成熟的工具

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/Relational-Databases-vs-HBase.png)

但是，随着越来越多的数据的出现，需要对数据库进行扩展。一种垂直扩展的方法是使用更大的服务器，但这会增加成本，而且随着数据大小的增加会有更多的限制。 

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/Relational-Databases-vs-HBase-2.png)

## NoSQL出现的原因

垂直扩展的另一种方法是使用一组机器进行水平扩展，这些机器可以使用普通的廉价硬件。这种方法更加低成本且可靠。要对RDBMS进行水平分区或切分，数据是基于行分布的，有些行存储在一台机器上，其他行存储在另外一些机器上，但是，对关系数据库进行分区或切分是很复杂的，而且它的设计初衷并不是为了自动化地完成这项工作。此外，因为数据的分散存储，还有可能造成事物一致性等特性的损失。关系数据库是为单个节点设计的，不是为在集群上运行而设计的。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/NoSQL-scale.png)

## 关系型数据库的局限性

传统数据库的规范化操作消除了冗余数据，从而提高了存储效率。但是在规范化模式中，查询操作往往需要将数据重新组合在一起，会导致表连接操作增加。然而HBase是不支持关系和连接的，在一起访问的数据也是存储在一起的，因此它避免了与关系模型相关的局限性。两种数据存储模型的差异如下图所示：

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/RDBMS-HBase-storage.png)

## HBase是为分布式、大规模和低延迟而设计的

按照key对数据进行分组对于在集群上运行的数据库是至关重要的。在水平分区或分片中，key的取值范围用于分片，它将不同的数据划分到多个服务器上，每个服务器上存储的是一个数据子集。HBase实际上是基于BigTable存储体系结构的一个实现，BigTable是由谷歌开发的分布式存储系统，用于管理结构化数据，可以扩展到非常大的数据规模。

HBase是一种面向列族的数据存储模式，同时它也是面向行的：每一行都由一个可以用于查找的键索引(例如，查找ID为1234的客户)。每个列族对相似的数据进行分组，而每一行可以看作是所有列族中所有值的联接。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/HBase-column.png)

HBase是一种分布式数据库。按键对数据进行分组是在集群上运行和分片的关键。键是更新操作的基本单元。分片将不同的数据分布在多个服务器上，每个服务器上都是数据子集。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/HBase-distributed-database.png)

## HBase数据模型

存储在HBase中的数据通过行键进行定位，行键类似于传统数据库中的主键。HBase中的记录按照行键进行顺序存储，这是HBase的基本原则，也是HBase模式设计中需要遵循的关键点。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/HBase-data-model.png)

表按照键的范围，划分为不同的行序列，称为regions。这种region会被分配到集群中的数据节点上，称为RegionServers。通过在集群中扩展region，数据库的读写能力也得到了扩展。这一切工作都是自动完成的，也是HBase实现水平分片的方式。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/HBase-tables.png)

下图显示了列族如何映射到存储文件。列族各自存储在单独的文件中，可以分别单独访问这些文件。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/HBase-column-families.png)

数据存储在HBase表的cell中。整个cell，包括附加的结构信息，称为键值(Key Value)。在每一个设定了Value的cell中存储了行键、列族名、列名、时间戳和值等信息。Cell中的Key是由行键、列族名、列名和时间戳组成的。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/Hbase-table-cells.png)

逻辑上，cell是以表的形式存储的。但实际的物理存储中，行是以cell的线性集合的形式来存储的，其中cell包括了行的所有键值信息。

在下图中，左上方显示的是数据的逻辑布局，右下方显示实际文件中的物理存储。列族是存储在各自单独的文件中的。在每一个设定了Value的cell中存储了行键、列族名、列名、时间戳和值等信息。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/Logical-vs-physical-storage.png)

如上文中所述，cell的value的完整坐标为：Table:Row:Family:Column:Timestamp ➔ Value。HBase表是稀疏填充的。如果某一列不存在数据，就不会被存储。表中的cell是已版本控制的无意义的字节数组，也可以使用时间戳或设置自己的版本控制系统。对于每一个row：family：Column 坐标，value可以有多个版本。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/sparse-data.png)

版本控制是HBase内置的。每次put操作既是insert(create)也是update，每个put都有自己的版本。Delete操作会得到一个墓碑标记。墓碑标记可以防止对应的数据在查询中被返回。可以通过参数来请求返回特定版本的数据。如果不指定任何参数，就会返回最新的版本。可以设置每个列族希望保留多少个版本的数据，默认为3个版本。当超过最大版本数时，会移除额外的记录。

![avatar](https://mapr.com/blog/hbase-and-mapr-db-designed-distribution-scale-and-speed/assets/blogimages/versioned-data.png)

