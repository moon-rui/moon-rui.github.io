---
layout: article
title: Windows环境下安装部署Spark
tags: Spark
---

> 虽然在实际的生产环境中，Spark都是部署在Linux环境中。但是对于Windows用户或者对Linux系统并不熟悉的用户来说，了解在Windows环境下的安装部署对Spark的入门学习还是有帮助的。
>

<!--more-->

进入官网下载地址<https://spark.apache.org/downloads.html>，这里选择`Pre-built for Apache Hadoop 2.7 and later `版本。

下载完成解压后，进入Spark所在目录的bin目录，通过`spark-shell`命令进入Spark环境。可以按照官网的[quick-start](https://spark.apache.org/docs/latest/quick-start.html)文档进行操作。

在通过`spark-shell`命令进入Spark环境的过程中，命令行中可能会抛出若干异常，这里给出相应的解决方案。

- 异常一：java.io.IOException: Could not locate executable null\bin\winutils.exe in the Ha

doop binaries.

这个问题是因为Windows中没有安装Hadoop环境，可以通过<https://github.com/steveloughran/winutils/blob/master/hadoop-2.7.1/bin/winutils.exe>下载winutils.exe文件来构建一个虚拟的Hadoop环境，并配置系统变量`HADOOP_HOME`。

**注意：**`winutils.exe`文件必须放在`HADOOP_HOME`路径的bin目录下。例如：配置`HADOOP_HOME`系统变量为`D:\Tools\hadoop`，则`winutils.exe`文件放在`D:\Tools\hadoop\bin`目录下。

- 异常二：java.lang.IllegalArgumentException: Error while instantiating 'org.apache.spark.

sql.hive.HiveSessionStateBuilder'

这个问题是`/tmp/hive `(如果Spark安装在D盘，则为`D:\tmp\hive`)目录权限导致的。可使用`winutils.exe`文件在Windows环境操作Linux命令来解决，具体过程如下：

```shell
D:\Tools\hadoop>winutils.exe ls d:\tmp\hive
d--------- 1 lenovo-PC\rui lenovo-PC\None 0 Nov 26 2018 d:\tmp\hive

D:\Tools\hadoop>winutils.exe chmod 777 d:\tmp\hive

D:\Tools\hadoop>winutils.exe ls d:\tmp\hive
drwxrwxrwx 1 lenovo-PC\rui lenovo-PC\None 0 Nov 26 2018 d:\tmp\hive
```
修复异常后，再运行`spark-shell`命令，可看到正常启动如下：

```shell
D:\spark-2.2.0-bin-hadoop2.7>.\bin\spark-shell
Spark context Web UI available at http://:4040
Spark context available as 'sc' (master = local[*], app id = local-
).
Spark session available as 'spark'.
Welcome to
      ____              __
     / __/__  ___ _____/ /__
    _\ \/ _ \/ _ `/ __/  '_/
   /___/ .__/\_,_/_/ /_/\_\   version 2.2.0
      /_/

Using Scala version 2.11.8 (Java HotSpot(TM) 64-Bit Server VM, Java 1.8.0_91)
Type in expressions to have them evaluated.
Type :help for more information.

scala>
```

Spark shell在初始化后会在当前命令行窗口创建名为“sc”的context和名为“spark”的session。可以从session中获得DataFrameReader，它可以把文本文件读取为一个数据集，其中每一行都作为数据集的一项读取。以下Scala命令创建名为“textFile”的数据集，然后对数据集运行count()、first()和filter()操作。 

```scala
scala> val textFile = spark.read.textFile("README.md")
textFile: org.apache.spark.sql.Dataset[String] = [value: string]

scala> textFile.count()
res0: Long = 103

scala> textFile.first()
res1: String = # Apache Spark

scala> val linesWithSpark = textFile.filter(line => line.contains("Spark"))
linesWithSpark: org.apache.spark.sql.Dataset[String] = [value: string]

scala> textFile.filter(line => line.contains("Spark")).count()
res2: Long = 20
```

map()、reduce()、collect()相关操作：

```scala
scala> textFile.map(line => line.split(" ").size).reduce((a, b) => if (a > b) a else b)
res3: Int = 22

scala> val wordCounts = textFile.flatMap(line => line.split(" ")).groupByKey(identity).count()
wordCounts: org.apache.spark.sql.Dataset[(String, Long)] = [value: string, count
(1): bigint]

scala> wordCounts.collect()
res4: Array[(String, Long)] = Array((online,1), (graphs,1), (["Parallel,1), (["Building,1), (thread,1), (documentation,3), (command,,2), (abbreviated,1), (overview,1), (rich,1), (set,2), (-DskipTests,1), (name,1), (page](http://spark.apache.org/documentation.html).,1),(["Specifying,1), (stream,1), (run:,1), (not,1), (programs,2), (tests,2), (./dev/run-tests,1), (will,1), ([run,1), (particular,2), (option,1), (Alternatively,,1), (by,1), (must,1), (using,5), (you,4), (MLlib,1), (DataFrames,,1), (variable,1), (Note,1), (core,1), (more,1), (protocols,1), (guidance,2), (shell:,2), (can,7), (site,,1), (systems.,1), (Maven,1), ([building, 1), (configure,1), (for,12), (README,1), (Interactive,2), (how,3), ([Configuration,1), (Hive,2), (system,1), (provides,1), (Hadoop-supported,1), (pre-built,1...
```



参考文档：<https://dzone.com/articles/working-on-apache-spark-on-windows>