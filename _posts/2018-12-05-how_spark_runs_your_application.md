---
layout: article
title: 【译】Spark应用运行机制
key: 2018-12-05_how_spark_runs_your_application
tags: Spark
---

> 关于Spark如何将数据集的转换和操作变为可执行的模型，介绍如何使用Spark Web UI查看任务执行信息
> 原文链接：[How Spark Runs Your Applications](https://mapr.com/blog/how-spark-runs-your-applications/)

<!--more-->

为了理解应用程序如何在集群上运行，了解数据集的转换非常重要，它分为两种类型，窄类型(narrow)和宽类型(wide)，在解释执行模型之前，我们将先讨论这两种类型。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image18.png)

## 窄转换与宽转换
数据转换是基于现有数据集创建一个新的数据集。当从现有数据集创建新数据集时，窄转换不必在分区之间移动数据。`filter`和`select`是两种典型的窄转换，在下面的示例中用这两个方法来检索航空公司“AA”的航班信息:
```scala
// select and filter are narrow transformations
df.select($"carrier",$"origin",  $"dest", $"depdelay", $"crsdephour").filter($"carrier" === "AA" ).show(2)

result:
+-------+------+----+--------+----------+
|carrier|origin|dest|depdelay|crsdephour|
+-------+------+----+--------+----------+
|     AA|   ATL| LGA|     0.0|        17|
|     AA|   LGA| ATL|     0.0|        13|
+-------+------+----+--------+----------+
```

在被称为pipelining的过程中，可以对内存中的数据集执行多个窄转换，从而使得窄转换的运行非常高效。

当创建新数据集时，在被称为shuffle的过程中，宽转换会导致数据在分区之间移动。数据通过网络发送到其他节点并写到磁盘，从而导致网络和磁盘IO，使shuffle成为一项成本高昂的操作。典型的宽转换包括`groupBy`、`agg`、`sortBy`和`orderBy`等。下面是一个按航空公司分组计算航班数量的宽转换。
```scala
df.groupBy("carrier").count.show
result:

+-------+-----+
|carrier|count|
+-------+-----+
|     UA|18873|
|     AA|10031|
|     DL|10055|
|     WN| 2389|
+-------+-----+
```
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image12.png)

## Spark执行模型
Spark执行模型可以分为三个阶段:创建逻辑计划(logical plan)，将其转换为物理计划(physical plan)，然后在集群上执行任务。

可以访问URL: <http://IP:4040>在浏览器中实时查看关于Spark作业的信息。对于已经完成的Spark应用程序，可以使用Spark history server在浏览器中查看信息，URL是:<http://IP:18080>。下面通过一些示例代码，详细介绍这三个阶段以及有关这些阶段的Spark UI信息。

### 逻辑计划
逻辑计划是在第一阶段被创建。该计划显示在应用操作时将执行哪些步骤。当对数据集应用转换时，将创建一个新的数据集。当这种情况发生时，新数据集将指向父数据集，从而产生定向无环图(directed acyclic graph，DAG)，用于Spark执行这些转换。

### 物理计划
第二阶段将逻辑DAG转换为物理执行计划。Spark Catalyst查询优化器为DataFrames创建物理执行计划，如下图所示:
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image9.png)
物理计划标识将会执行计划的资源，例如内存分区和计算任务。

### 查看逻辑和物理计划
通过调用explain(true)方法，可以看到数据集的逻辑和物理计划。在下面的代码中，我们看到df2的DAG由一个`FileScan`、一个针对`depdelay`的`Filter`和一个`Project`(选择列)组成。[数据集下载链接](https://pan.baidu.com/s/1AM6pBRBysj8sk3oc9oFezQ)
```scala
import org.apache.spark.sql.types._
import org.apache.spark.sql._
import org.apache.spark.sql.functions._

var file = "maprfs:///data/flights20170102.json"

case class Flight(_id: String, dofW: Long, carrier: String, origin: String, dest: String, crsdephour: Long, crsdeptime: Double, depdelay: Double,crsarrtime: Double, arrdelay: Double, crselapsedtime: Double, dist: Double) extends Serializable

val df = spark.read.format("json").option("inferSchema", "true").load(file).as[Flight]

val df2 = df.filter($"depdelay" > 40)

df2.take(1)

result:
Array[Flight] = Array(Flight(MIA_IAH_2017-01-01_AA_2315, 7,AA,MIA,IAH,20,2045.0,80.0,2238.0,63.0,173.0,964.0))

df2.explain(true)

result:
== Parsed Logical Plan ==
'Filter ('depdelay > 40)
+- Relation[_id#8,arrdelay#9,…] json

== Analyzed Logical Plan ==
_id: string, arrdelay: double…
Filter (depdelay#15 > cast(40 as double))
+- Relation[_id#8,arrdelay#9…] json

== Optimized Logical Plan ==
Filter (isnotnull(depdelay#15) && (depdelay#15 > 40.0))
+- Relation[_id#8,arrdelay#9,…] json

== Physical Plan ==
*Project [_id#8, arrdelay#9,…]
+- *Filter (isnotnull(depdelay#15) && (depdelay#15 > 40.0))
   +- *FileScan json [_id#8,arrdelay#9,…] Batched: false, Format: JSON, Location: InMemoryFileIndex[maprfs:///..],
```
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image11.png)

在web UI SQL页面上可以查看Catalyst生成的计划的更多细节(http://IP:4040/SQL/)。单击描述链接将显示查询操作的DAG和详细信息。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image19.png)

在下面的代码中，我们看到df3的物理计划由`FileScan`, `Filter`, `Project`, `HashAggregate`, `Exchange`, 和`HashAggregate`组成。其中的`Exchange`是由`groupBy`转换引起的shuffle操作。在对Exchange中的数据进行shuffle之前，Spark对每个分区执行哈希聚合。在交换之后，将对以前的子聚合再进行散列聚合。注意，如果df2被缓存，将在DAG中进行内存扫描，而不是文件扫描。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image6.png)

在Web UI中单击该查询的SQL描述链接将显示下面的DAG。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image5.png)

### 在集群上执行任务
第三阶段，任务在集群上排定并执行。调度器根据转换将作业(Job)划分为多个阶段(Stage)。窄转换(没有数据移动的转换)将被分组(pipe-lined)到一个阶段。这个示例的物理计划有两个阶段，在第一个阶段中包含交换之前的所有内容。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image16.png)

每个阶段由基于数据集分区的任务(Task)组成，这些任务将并行执行相同的计算。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image17.png)

调度器将阶段任务集合提交给任务调度器，任务调度器通过集群管理器启动任务。这些阶段是按顺序执行的，当作业的最后一个阶段完成时，就认为操作已经完成。在创建新数据集时，这种执行序列会多次出现。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image4.png)

以下是执行过程中各个组件的总结摘要:
- **Task: ** 在单个机器节点上运行的执行单元
- **Stage: ** 基于输入数据分区的一组任务，并行执行相同的计算
- **Job: ** 有一个或多个Stage
- **Pipelining: ** 当数据集转换操作可以在不移动数据的情况下进行计算,将数据集压缩到单个阶段
- **DAG: ** 数据集转换的逻辑图

### 在Web UI上查看任务执行
下面是运行上述代码后web UI Jobs选项页的屏幕截图。该页面展示了运行中的和最近完成的Spark作业的详细执行信息。提供作业的性能表现以及作业、阶段和任务的运行进度。在本例中，作业Id 2是由df3上的collect操作触发的作业。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image2.png)

单击Jobs页面上的Description列中的链接，就会转到Job Details页面。该页面详细介绍了作业、阶段和任务的进展。我们看到这个作业由2个阶段组成，shuffle前的阶段有2个任务，shuffle后阶段有200个任务。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image15.png)

任务数量与分区相对应：第一阶段读取文件后，有2个分区；shuffle后，分区的默认数目是200。可以使用`rdd.partitions.size`方法查看数据集上的分区数量，如下所示。
```scala
df3.rdd.partitions.size
result: Int = 200

df2.rdd.partitions.size
result: Int = 2
```

在stage选项页下，可以通过单击description列中的stage链接来查看stage的详细信息。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image8.png)

这里我们有任务的摘要度量和聚合度量，以及执行器的聚合度量。可以使用这些度量来定位执行程序或任务分布中的问题。如果任务流程时间分配不均衡，那么资源可能会被浪费。

Storage选项页提供关于持久化数据集的信息。如果在数据集上调用Persist或Cache，则该数据集将被持久化，以便后续在该数据集上执行计算操作。此页面将展示缓存了数据集底层RDD的哪个部分以及缓存在各种存储介质中的数据量。可以看到重要的数据集是否被存入内存。还可以单击链接查看关于持久数据集的更多详细信息。如果不再需要缓存的数据集，可以调用`Unpersist`来取消缓存。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image7.png)

尝试缓存df2，执行一个操作，然后在storage选项页上查看它如何持久化，以及在job details页面上查看它如何影响df3的计划和执行时间。注意，缓存后执行时间更快。
```scala
df2.cache
df2.count
df3.collect
```

注意，在job4中，当缓存df2并再次执行df3 collect时，会跳过第一个阶段。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image14.png)

Environment选项页列出了Spark应用程序环境的所有属性。如果想查看启用了哪些配置，可以访问此页面。只有通过spark-defaults.conf、SparkSession或命令行指定的值才会显示在这里。对于所有其他配置属性，使用的是默认值。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image3.png)

在executor选项页下，可以看到每个executor的处理和存储:
- **Shuffle Read Write Columns: ** 显示各阶段之间传输的数据的大小
- **Storage Memory Column: ** 显示当前已用和可用内存
- **Task Time Column: ** 显示任务时间和垃圾收集时间

使用此页面可以确认应用程序拥有所期望的资源数量。可以通过单击thread dump链接查看线程调用堆栈。
![avatar](https://mapr.com/blog/how-spark-runs-your-applications/assets/image20.png)

## 总结
在本文中，我们讨论了Spark执行模型，并熟悉了如何在Spark Web UI上查看任务的执行信息。了解Spark如何运行应用程序，对于调试、分析和调优应用程序性能来说是非常重要的。