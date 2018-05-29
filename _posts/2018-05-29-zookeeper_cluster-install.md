---
layout: post
title: Zookeeper集群安装配置
key: 2018-05-29-zookeeper_cluster_install
tags: 
   - Zookeeper
   - 分布式
---

## 1. 环境说明

本次安装部署使用三台主机作为集群，系统为CentOS 6.5 64位。/etc/hosts的配置信息如下：

```
192.168.196.12 moon111
192.168.196.29 moon112
192.168.196.11 moon113
```

需要准备好JDK环境，这里所用的版本为jdk1.8.0_121。Zookeeper所安装的版本为3.4.10。

## 2. 安装

官网下载地址https://archive.apache.org/dist/zookeeper/

下载完成后解压安装包：

```shell
[hadoop@moon111 ~]$ tar -zxf zookeeper-3.4.10.tar.gz 
```

## 3. 配置

解压完成后具体配置操作如下：

```shell
[hadoop@moon111 ~]$ cd zookeeper-3.4.10/conf/
[hadoop@moon111 conf]$ cp zoo_sample.cfg zoo.cfg
```

编辑配置文件zoo.cfg：

```properties
tickTime=2000
initLimit=10
syncLimit=5
dataDir=/home/hadoop/zookeeper-3.4.10/data
dataLogDir=/home/hadoop/zookeeper-3.4.10/log
clientPort=2181
server.1=moon111:2888:3888
server.2=moon112:2888:3888
server.3=moon113:2888:3888
```

其中，clientPort 为 zookeeper的服务端口   server.1、server.2、server.3 为 zk 集群中三个 node 的信息，定义格式为  hostname:port1:port2，其中 port1 是 node 间通信使用的端口，port2 是node  选举使用的端口，需确保三台主机的这两个端口都是互通的。 

另外，dataDir 和 dataLogDir 需要在启动前创建完成：

```shell
[hadoop@moon111 conf]$ mkdir /home/hadoop/zookeeper-3.4.10/data
[hadoop@moon111 conf]$ mkdir /home/hadoop/zookeeper-3.4.10/log
```

更改日志配置

Zookeeper 默认会将控制台信息输出到启动路径下的 zookeeper.out 中，通过如下方法，可以让 Zookeeper 输出按尺寸切分的日志文件： 

1）修改conf/log4j.properties文件，修改zookeeper.root.logger配置项

```properties
zookeeper.root.logger=INFO, CONSOLE
# 修改为
zookeeper.root.logger=INFO, ROLLINGFILE
```

2）修改bin/zkEnv.sh文件，修改ZOO_LOG4J_PROP配置项

```shell
ZOO_LOG4J_PROP="INFO,CONSOLE"
# 修改为
ZOO_LOG4J_PROP="INFO,ROLLINGFILE"
```

配置完成后，将安装目录复制到另两台主机上：

```shell
[hadoop@moon111 bin]$ scp -r /home/hadoop/zookeeper-3.4.10 moon112:/home/hadoop/
[hadoop@moon111 bin]$ scp -r /home/hadoop/zookeeper-3.4.10 moon113:/home/hadoop/
```

复制完成后，分别在三台主机的 dataDir 路径下创建一个文件名为 myid 的文件，文件内容为该 zk 节点的编号。例如，在第一台主机上建立的 myid 文件内容是 1，第二台是 2，第二台是 3。

## 4. 启动

进入每台主机zookeeper的bin目录下，执行如下命令启动：

```shell
[hadoop@moon111 bin]$ ./zkServer.sh start
```

三台主机启动完毕后，可执行如下命令查看状态：

```shell
[hadoop@moon111 bin]$ ./zkServer.sh status
```

返回结果如下：

```shell
[hadoop@moon111 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
[hadoop@moon112 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: leader
[hadoop@moon113 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
```

可以看出，3个节点中，有1个leader和2个follower。

## 5. 测试集群高可用性

停掉集群中为leader的节点，例如在这里为moon112：

```shell
[hadoop@moon112 bin]$ ./zkServer.sh stop
```

查看moon111和moon113的状态：

```shell
[hadoop@moon111 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
[hadoop@moon113 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: leader
```

可以看到此时113节点成为了集群中的leader。

再次启动112节点的zookeeper，并查看状态：

```shell
[hadoop@moon112 bin]$ ./zkServer.sh status
ZooKeeper JMX enabled by default
Using config: /home/hadoop/zookeeper-3.4.10/bin/../conf/zoo.cfg
Mode: follower
```

此时，112成为了集群中的follower。

## 6. 参考资料

Zookeeper-3.4.9 集群搭建     https://blog.csdn.net/chenshijie2011/article/details/75669942

centos6.5环境下zookeeper-3.4.6集群环境部署及单机部署详解    https://blog.csdn.net/reblue520/article/details/52279486