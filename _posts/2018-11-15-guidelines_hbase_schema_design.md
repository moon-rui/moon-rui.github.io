---
layout: article
title: [译] HBase表模式设计指南
tags: HBase
---

> 一片英文博客，讲解得较为详细，翻译出来以供参考。
> 原文链接：[Guidelines for HBase Schema Design](https://mapr.com/blog/guidelines-hbase-schema-design/)

<!--more-->

这篇文章主要讨论HBase和传统关系型数据库在表模式设计上的区别，并对合适的HBase表设计给出一些指南。

**关系型数据库 vs. HBase**

HBase与关系型数据库之间并不存在一对一的映射关系。在关系型的模型设计中，关注的重点在于如何描述一个实体及与其他实体间的关系，而查询和索引是之后再考虑的设计事项。

但在HBase中，模式设计是“查询优先的”。首先需要考虑到所有可能的查询，进而再设计相应的模型。在设计模型的同时，应该利用到HBase自身的优势。考虑数据的访问模式，使得在你的模型中，一起读取的数据也是存储在一起的。HBase是为集群而设计的，记住这一点很重要。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post1.png)

**规范化**

在关系型数据库中，通过将重复信息放在各自的表中来消除冗余，实现模式的规范化。这样做的好处有：

- 在做更新操作时不必更新多个副本，可以加快写操作的速度。
- 通过使用单个副本而不是多个副本来减少存储空间的占用。

然而，这会导致更多的表连接操作。数据要从多个表中检索，进而导致查询操作的耗时增加。

在下图的例子中，订单表和订单项目表是一对多的关系，订单表的外键ORDER_ID与订单项目表中的字段相对应。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post2.png)

**反规范化**

在反规范化的数据存储中，一张表存储了关系中的多个索引，可以看作是对表连接的替代。通常在使用HBase时，以反规范化或复制数据的方式使得访问的数据存储在一起。

**父子关系嵌套实体**

这里给出一个HBase中反规范化的例子，如果数据表中存在一对多的关系，可以在HBase中将其建模为一行。在下图的例子中，订单和相关联的项目存储在一起，通过行键可以一次性读取。和表关联操作相比，这样存储大大提高了读操作的效率。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post3.png)

行键对应父实体的ID，也就是订单ID。订单数据和订单项目数据各有一个列族。订单项是嵌套的，订单项ID放入列名中，其他所有非标识属性放入值中。 

当获得子实体的唯一方法是通过父实体时，这种模式设计方法是合适的。

 **关系型数据库中的多对多关系**

这里给出一个多对多关系在关系型数据库中的例子。数据查询要求如下：

- 某个用户的名字
- 某本书的标题
- 某个用户所有的评价
- 所有用户对某本书的评价

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post4.png)

**HBase中的多对多关系**

对于实体表，通常有一个列族存储所有实体属性，另外的列族存储到其他实体的链接。如下图所示：

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post5.png)

**通用数据、事件数据和实体属性值**

无模式的通用数据通常表示为名称值或实体属性值。在关系型数据库中，这种数据表示起来很复杂。组成传统关系数据表的属性列和表中的每一行都是相关的，因为每一行都表示着一个相似对象的实体。不同的属性集代表不同类型的对象，因此属于不同的表。HBase的优点是可以动态定义列，在列限定符中放入属性名，并按列族对数据进行分组。 

这里给出一个临床病人事件数据的例子。行键是患者ID加上时间戳。可变的事件类型放在列限定符中，事件度量放在列值中。OpenTSDB是可变系统监视数据的一个例子。 

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post6.png)

**HBase中的自连接关系**

自连接关系指的是两个匹配字段定义在同一张表中。

考虑一个Twitter关系的模型，查询需求为：用户A关注了哪些用户，哪些用户关注了用户A。这里给出一个可能的解决方案：将用户ID放入复合行键中，关系类型作为分隔符。比如，Carol关注了Steve Jobs，同时BillyBob关注了Carol。这样一来就允许对每个人进行行键扫描：carol:follows or carol:followedby。

下图是Twitter关系表的一个示例：

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post7.png)

**树形、图形数据**

下面是一个邻接表的示例，其中每个父级和子级节点使用要给单独的列：

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post8.png)

每一行表示一个节点，行键等于节点ID。父级节点有一个列族P，子级节点有一个列族C。列限定符等于父节点或子节点id，值等于节点的类型。这样就可以通过行键快速找到父节点或子节点。

**继承映射**

在下图的在线商店示例中，产品的类型是行键中的前缀。根据产品类型的不同，有些列也是不同的，还有可能为空值。这样可以在同一个表中对不同的产品类型进行建模，并方便地按产品类型进行扫描。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post9.png)

**数据访问模式**

1. 使用案例：大规模离线ETL分析和生成派生数据

在分析学中，写数据的次数比读数据的次数要多出几个数量级。离线分析可用于提供在线查看的快照，同时离线系统没有低延迟的需求。因此，离线HBase ETL数据访问模式(如Map Reduce或Hive)具有高延迟读取和高吞吐量写入的特点。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post10.png)

2. 使用案例：物化视图

为了给在线网站提供快速读取，或通过数据分析提供在线数据视图，MapReduce可以为不同的读取需求或物化视图将数据做不同的分组。批量离线分析还可以用于为在线视图提供快照。这会是高吞吐量的离线批量写和高延迟的读取(在线)。

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post11.png)

例子包括：

- 生成派生数据、复制数据用于在HBase模式中的读取以及延迟二级索引 

模式设计探究：

- 来自HDFS或HBase的原始数据
- 用于从原始数据进行数据转换和ETL的MapReduce。
- 从MapReduce到HBase的批量导入
- 提供HBase在线读取数据服务

为读取而设计模型意味着积极地反规范化数据，以便将一起读取的数据存储在一起。

3. Lambda架构

Lambda体系结构通过将问题分解为三个层次，即批处理层(batch layer)、服务层(serving layer)和快速处理层(speed layer)，解决了对任意数据实时计算任意函数的问题。 

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post12.png)

MapReduce任务用于规模化地创造对消费者有用的工件。通过在Storm集群中处理对HBase的更新来实现对增量更新的实时处理，并应用于MapReduce作业生成的工件。

批处理层预先计算批处理视图。在批处理视图中，结果从预计算视图中读取。预计算视图建立了索引，以支持随机读取快速访问。服务层对批处理视图进行索引并将其加载，使其可以高效查询以便从视图中获取特定的值。服务层数据库只需要批量更新和随机读取。每当批处理层完成预计算批处理视图时，服务层就会执行更新操作。

在该架构中，可以使用Storm进行基于流的处理，使用Hadoop进行批处理。快速处理层仅生成基于最新数据的视图，并用于在批处理未覆盖的近几个小时内的数据进行计算的函数。为了实现最低的延迟，快速处理层不会同时查看所有的新数据。相反，它在接收新数据时更新实时视图，而不是像批处理层那样重新计算它们。在快速处理层，HBase为Storm提供了持续增加实时视图的能力。

需要在HBase中设置工作标记，以便让Storm知道哪些是需要处理的新数据。处理组件会在进入系统时扫描这些标记并做处理。

**MapReduce执行过程和数据流**

![avatar](https://mapr.com/blog/guidelines-hbase-schema-design/assets/blogimages/Hbase-post13.png)

MapReduce执行中的数据流如下所示：

1. 数据从Hadoop文件系统加载
2. 作业定义数据的输入格式
3. 数据被分割到在所有节点上运行的不同map()方法上
4. record readers将数据解析为键-值对，作为map()方法的输入
5. map()方法将生成的键-值对发送给partitioner
6. 当有多个reduce任务时，mapper为每个reduce任务创建一个分区
7.  键值对在每个分区中按键排序
8. reduce()方法获取中间结果的键-值对，并将其规约为键-值对的最终结果列表
9. 作业定义数据的输出格式
10. 数据被写入Hadoop文件系统