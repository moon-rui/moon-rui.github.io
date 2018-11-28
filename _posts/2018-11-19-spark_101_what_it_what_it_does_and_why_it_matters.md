---
layout: article
title: 【译】Spark简介
tags: Spark
---

> 关于Spark的简介，包括发展历史、应用领域等
>
> 原文链接：[Spark 101: What Is It, What It Does, and Why It Matters](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/)

<!--more-->

## Spark是什么

Spark是一种通用的分布式数据处理引擎，适用于多种环境。在Spark的核心数据处理引擎之上，有用于SQL、机器学习、图形计算和流处理的库，并且可以在应用程序中一起使用。Spark支持的编程语言包括：Java、Python、Scala和R。应用程序开发人员和数据科学家将Spark整合到他们的应用程序中，用以快速并大规模地查询、分析和转换数据。最常见的应用领域包括大型数据集间的ETL和SQL批处理任务、来自传感器，物联网或金融系统的流数据处理以及机器学习任务。

![avatar](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/assets/image7.png)

## 历史

为了进一步理解Spark，了解它的发展历史是很有必要的。在Spark出现之前，弹性的分布式处理框架MapReduce，使谷歌能够在大型服务器集群上索引网络中爆炸式增长的内容。

![avatar](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/assets/image6.png)

谷歌的数据处理策略中有三个核心概念：

1. **分布式数据：**当数据文件上传到集群中时，它被分割成块(称为数据块)，分布在数据节点中，并在集群中复制。

2. **分布式计算：**用户指定一个map函数和reduce函数，map函数处理键/值对，并生成一组中间键/值对，reduce函数合并包含相同键的中间键/值对。以这种功能风格编写的程序自动实现了并行化，并通过以下方式在大型商品机器集群上执行：

   - mapping进程在分配的数据节点上运行，只处理其所在块上的来自分布式文件的数据。
   - mapping进程产生的数据结果以“shuffle and sort”的方式发送给reducer：来自mapper的键/值对按键进行排序，按reducer进行划分，然后通过网络发送并写入到reducer节点上按键排序的“序列文件(sequence files)”。
   - reducer进程在分配的节点上执行，并且只工作在它的数据子集(序列文件)上。reducer进程的输出结果会被写入到一个输出文件中。

3. **容错性：**通过转移到另一个节点进行数据处理来保证容错性。

## MapReduce word count执行示例

![avatar](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/assets/image4.png)

一些迭代算法，比如PageRank，谷歌用在其搜索引擎结果中对网站进行排名。这种算法需要将多个MapReduce作业链接在一起，这会导致对磁盘的大量读写操作。当多个MapReduce任务链接在一起时，对于每个MapReduce任务，数据从一个分布式文件块读取到一个map进程中，从中间的序列文件写入和读取，然后从reducer进程写入到输出文件中。

![avatar](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/assets/image3.png)

在谷歌发表了一份白皮书描述MapReduce框架(2004)的一年之后,Doug Cutting和Mike Cafarella创造了Apache Hadoop™。

Apache Spark™创造于2009年，一开始是作为加州大学伯克利分校AMPLab的一个项目。Spark于2013年成为Apache软件基金会的孵化项目，2014年初被提升为基金会的顶级项目之一。 Spark目前是基金会管理的最活跃的项目之一，围绕该项目成长起来的社区既包括多产的个人贡献者，也包括资金充裕的企业支持者，如Databricks、IBM和华为。

Spark项目的目标是保持MapReduce的可伸缩、分布式、容错处理框架的优点，同时使其更高效、更易于使用。相比MapReduce，Spark的优点是：

- Spark通过将并行操作的数据缓存在内存中获得了更快的执行速度，而MapReduce的执行涉及了更过的磁盘读写操作。
- Spark在JVM进程内部运行多线程任务，而MapReduce是作为更重量级的JVM进程运行。这使得Spark启动速度更快，并行性更好，CPU利用率更高。
- Spark提供了比MapReduce更丰富的函数式编程模型。
- Spark非常适合使用迭代算法并行处理分布式数据的领域。 

## Spark应用程序是如何在集群上运行的

下图示例中是一个在集群上运行的Spark应用程序。

- Spark应用程序作为独立的进程运行，由driver程序中的SparkSession对象进行协同。
- 资源或集群管理器为Worker分配任务，每个分区分配一个任务。
- 每个任务将其工作单元应用于分区中的数据集，并输出新的分区数据集。因为迭代算法重复地对数据进行操作，所以它们可以从迭代缓存数据集中获益。
- 任务处理结果被发送回driver程序或保存到磁盘。

![avatar](https://mapr.com/blog/spark-101-what-it-what-it-does-and-why-it-matters/assets/image1.png)

Spark支持以下资源/集群管理器：

- **Spark Standalone** - 包含在Spark中的一个简单集群管理器
- **Apache Mesos** - 一个通用的集群管理器，支持Hadoop应用程序
- **Apache Hadoop YARN** - Hadoop 2.0版本中的资源管理器
- **Kubernetes** - 用于自动化部署、扩展和管理容器化应用程序的开源系统

Spark还有本地模式，其中driver程序和executors作为线程在计算机上运行，而不是作为集群运行，用于在本地开发Spark应用程序。

## Spark能做什么？

Spark能够处理分布在数千个的物理或虚拟服务器集群中的PB级数据。它拥有大量的开发人员库和API，并支持Java、Python、R和Scala等语言，其灵活性使它适用于广泛的应用领域。Spark常用于分布式数据存储，如MapR-XD、Hadoop的HDFS和Amazon的S3，以及流行的NoSQL数据库，如MapR-DB、Apache HBase、Apache Cassandra和MongoDB，以及分布式消息存储，如MapR-ES和Apache Kafka。

典型的应用包括：

**信息流处理：**从日志文件到传感器数据，应用程序开发人员越来越需要处理流式数据。这些数据以稳定流的形式获得，通常同时会有多个来源。将这些数据流存储在磁盘上并进行做后续分析当然是可行的，但是有些时候在得到数据时就对其进行处理是更加合适的操作。例如，实时处理与金融交易相关的数据流，以识别并杜绝潜在的欺诈交易。

**机器学习：**随着数据量的增长，机器学习方法变得更加可行和精确。在将相同的解决方案应用于新的未知的数据之前，可以通过已知的数据集训练软件，以识别数据并对触发器进行操作。Spark能够将数据存储在内存中并快速运行重复查询，这使得它成为训练机器学习算法的极佳选择。多次大规模地运行类似的查询，可以显著地减少为找到最有效的算法而遍历一组可能的解决方案所需的时间。

**交互分析：**与运行预定义的查询来创建销售、生产率或股价的静态报表相比，业务分析师和数据科学家更希望通过提问或查看结果的方式来探索数据，然后稍微修改初始问题，或者进一步深入研究结果。 这种交互式查询过程需要如Spark这种能够快速响应和适应的系统。

**数据整合：**跨业务的不同系统生成的数据很少能够简单且容易地组合在一起进行报告或分析。提取、转换和加载(ETL)流程通常用于从不同系统中提取数据，对其进行清理和标准化，然后将其加载到单独的系统中进行分析。Spark(和Hadoop)越来越多地被用于ETL过程中以降低数据整合所需的成本和时间。

## 谁在使用Spark？

许多技术供应商都迅速地支持了Spark，他们认识到有机会将现有的大数据产品扩展到Spark提供的真正价值的领域中，比如交互式查询和机器学习。IBM和华为等知名企业已经在这项技术上投入了大量资金，越来越多的初创企业正在建立完全或部分依赖Spark的业务。例如，在2013年，负责创建Spark的Berkeley团队创建了Databricks，它提供了一个由Spark支持托管的端到端数据平台。该公司资金雄厚，在2013年、2014年、2016年和2017年的四轮投资中获得了2.47亿美元，并且Databricks员工在改进和扩展Apache Spark项目的开源方面继续发挥着突出作用。

主要的Hadoop商业版供应商，包括MapR、Cloudera和Hortonworks，都在现有产品的基础上支持基于YARN的Spark，努力为客户增加价值。IBM、华为和其他公司都对Apache Spark进行了重大的投资，将其集成到自己的产品中，并为Apache项目提供改进和扩展功能。搜索引擎百度、电子商务平台淘宝和社交网络公司腾讯等互联网公司都在大规模运行基于Spark的业务，据报道，腾讯的8亿活跃用户每天产生超过700 TB的数据，通过8000多个计算节点的集群进行处理。

除了这些互联网巨头，制药公司诺华(Novartis)依赖Spark来减少将建模数据送到研究人员手中所需的时间，同时确保伦理和契约上得到保障。

## 是什么让Spark与众不同？

选择Spark的原因很多，但以下三点是关键：

**简单：**Spark提供了丰富的API，这些API都是专门为快速轻松地与大规模数据交互而设计的。

**快速：**Spark的设计初衷就是为了快速计算，在内存和磁盘上都可以运行。在2014年的Daytona GraySort基准测试挑战赛中，Databricks的一个团队与加州大学圣地亚哥分校的一个团队凭借使用Spark获得并列第一的成绩(<https://spark.apache.org/news/spark-wins-daytona-gray-sort-100tb-benchmark.html>)。挑战赛内容是处理静态数据集：Databricks团队仅用23分钟就可以处理存储在固态硬盘上的100tb数据，而上一位获奖者使用Hadoop和不同的集群配置，耗时72分钟。在对存储在内存中的数据做交互式查询时，Spark的性能表现更好。在这些情况下，Spark可以比Hadoop的MapReduce快100倍。

**支持：**Spark支持一系列编程语言，包括Java、Python、R和Scala。Spark支持与Hadoop生态系统内外的许多存储解决方案进行紧密集成，包括：MapR(文件系统、数据库和事件存储)、Apache Hadoop (HDFS)、Apache HBase和Apache Cassandra。此外，Apache Spark社区是一个大型的、活跃的、国际化的社区。越来越多的商业提供者，包括Databricks、IBM和所有主要Hadoop供应商，提供了对基于Spark的解决方案的全面支持。

## 数据管道的力量

Spark的强大之处在于，它能够将迥然不同的技术和流程组合成一个统一、连贯的整体。在Spark出现之前，选择数据、以各种方式转换数据和分析转换结果的独立任务可能需要一系列单独的处理框架，比如Apache Oozie。Spark提供了将这些功能组合在一起的能力，跨越批处理、流和交互工作流之间的边界，以提高用户的工作效率。

Spark作业在内存中连续执行多个操作，仅在内存容量限制需要时才溢出到磁盘。Spark简化了这些不同流程的管理，提供了一个集成的整体——即一个更容易配置、更容易运行、更容易维护的数据管道。在像ETL这样的使用场景中，这些管道会变得非常复杂，将大量的输入和处理步骤组合成一个统一的整体，并交付期望的结果。