---
layout: article
title: Hadoop集群安装配置
key: 2018-05-29-hadoop_cluster_install
tags: Hadoop
---

> CentOS 6.5 Hadoop 2.7.6

<!--more-->

## 1. 环境说明

本手册以CentOS 6.5 64位作为系统环境，基于Hadoop 2.7.6版本。使用10.0.12.111，10.0.12.112，10.0.12.113三个节点作为集群环境，其中111为master节点，其余两个作为slave节点。

## 2. 流程介绍

Hadoop 集群的安装配置大致为如下流程:

1. 选定一台机器作为 Master
2. 在 Master 节点上配置 hadoop 用户、安装 SSH server、安装 Java 环境
3. 在 Master 节点上安装 Hadoop，并完成配置
4. 在其他 Slave 节点上配置 hadoop 用户、安装 SSH server、安装 Java 环境
5. 将 Master 节点上的 hadoop 目录复制到其他 Slave 节点上
6. 在 Master 节点上开启 Hadoop


## 3. Master节点安装Hadoop环境

### 3.1 创建Hadoop用户

以root用户登录服务器后，执行命令创建新用户hadoop并修改密码：

```shell
useradd -m hadoop -s /bin/bash   # 创建新用户hadoop
passwd hadoop   # 修改密码
```

接下来在命令行中执行`visudo`命令，为hadoop用户增加管理员权限，方便部署。

执行`visudo`命令后，找到 `root   ALL=(ALL)    ALL` 这行（应该在第98行，不同系统可能有所区别 ），然后在这行下面增加一行内容：`hadoop   ALL=(ALL)   ALL` （当中的间隔为tab），如下所示：

```
##
## The COMMANDS section may have other options added to it.
##
## Allow root to run any commands anywhere
root    ALL=(ALL)       ALL
hadoop  ALL=(ALL)       ALL		## 增加该行
```

修改完毕后保存退出，使用刚创建的hadoop用户登录服务器。

### 3.2 安装SSH

Hadoop集群和单节点模式都需要用到 SSH 登陆（类似于远程登陆，你可以登录某台 Linux 主机，并且在上面运行命令），一般情况下，CentOS 默认已安装了 SSH client、SSH server，打开终端执行如下命令进行检验：

```shell
rpm -qa | grep ssh
```

如果返回的结果如下所示，包含了 SSH client 跟 SSH server，则不需要再安装。

```shell
[hadoop@cetiti111 ~]$ rpm -qa | grep ssh
libssh2-1.4.2-1.el6.x86_64
openssh-5.3p1-94.el6.x86_64
openssh-server-5.3p1-94.el6.x86_64
openssh-clients-5.3p1-94.el6.x86_64
```

若需要安装，则可以通过 yum 进行安装：

```shell
sudo yum install openssh-clients
sudo yum install openssh-server
```

### 3.3 安装JDK

执行`java -version`命令，若提示commend not found，表名未安装JDK环境。

使用root用户登录，进入/usr目录，在该目录下创建java安装目录（可根据需求选择安装目录）

```shell
[root@cetiti111 .ssh]$ cd /usr
[root@cetiti111 usr]$ mkdir java
```

将jdk安装包拷贝到安装目录下并进行解压，使用FTP工具将jdk安装包（示例：jdk-8u121-linux-x64.tar.gz）文件上传到服务器/usr/java目录下。解压jdk安装包，解压后在/usr/java目录下得到相应文件夹（示例：文件夹jdk1.8.0_121），解压命令如下：

```shell
[root@cetiti111 usr]$ cd java     # 进入jdk安装包所在目录
[root@cetiti111 java]$ tar -zxvf jdk-8u121-linux-x64.tar.gz    # 解压
```

编辑/etc文件夹下的profile文件，配置环境变量

```shell
[root@cetiti111 java]$ vi /etc/profile # 打开profile文件
```

找到下面相应的配置内容，进行修改（如果没有相应配置内容，则手动添加）：

```
export JAVA_HOME=/usr/java/jdk1.8.0_121  #路径修改为解压后的jdk文件夹路径
export JRE_HOME=$JAVA_HOME/jre
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar
export PATH=$JAVA_HOME/bin:$JRE_HOME/bin:$PATH
```

修改完毕后保存退出，执行命令 `source /etc/profile`使配置文件生效。

再次使用hadoop用户登录，执行`java -version`命令，会输出java的版本信息：

```shell
[hadoop@cetiti111 ~]$ java -version
java version "1.8.0_121"
Java(TM) SE Runtime Environment (build 1.8.0_121-b13)
Java HotSpot(TM) 64-Bit Server VM (build 25.121-b13, mixed mode)
```

### 3.4 安装Hadoop

可以访问<http://hadoop.apache.org/releases.html>网址选择相应的版本下载Hadoop，这里选择的是2.7.6版本。需要注意的是，下载时请下载binary格式即 **hadoop-2.x.y.tar.gz** 这个格式的文件，这是编译好的，另一个包含 src 的则是 Hadoop 源代码，需要进行编译才可使用。

下载时强烈建议也下载 **hadoop-2.x.y.tar.gz.mds** 这个文件，该文件包含了检验值可用于检查 hadoop-2.x.y.tar.gz 的完整性，否则若文件发生了损坏或下载不完整，Hadoop 将无法正常运行。

将下载好的两个文件上传至/usr/local目录下(可根据需求选择安装目录)，执行如下命令：

```shell
[hadoop@cetiti111 ~]$ cd /usr/local/
[hadoop@cetiti111 local]$ cat hadoop-2.7.6.tar.gz.mds | grep 'MD5'
hadoop-2.7.6.tar.gz:    MD5 = 6B DF 41 74 20 2C 82 77  4F 31 47 2E 04 3D 61 07
[hadoop@cetiti111 local]$ md5sum hadoop-2.7.6.tar.gz | tr "a-z" "A-Z"
6BDF4174202C82774F31472E043D6107  HADOOP-2.7.6.TAR.GZ
```

若文件不完整则这两个值一般差别很大，可以简单对比下前几个字符跟后几个字符是否相等即可，如果两个值不一样，请务必重新下载。

文件完整性校验完毕后，解压文件：

```shell
[hadoop@moon111 local]$ sudo tar -zxf hadoop-2.7.6.tar.gz		# 解压文件
[hadoop@moon111 local]$ sudo chown -R hadoop ./hadoop-2.7.6	# 修改文件权限
```

Hadoop 解压后即可使用。输入如下命令来检查 Hadoop 是否可用，成功则会显示 Hadoop 版本信息：

```shell
[hadoop@moon111 local]$ cd /usr/local/hadoop-2.7.6/bin
[hadoop@moon111 bin]$ ./hadoop version
Hadoop 2.7.6
Subversion https://shv@git-wip-us.apache.org/repos/asf/hadoop.git -r 085099c66cf28be31604560c376fa282e69282b8
Compiled by kshvachk on 2018-04-18T01:33Z
Compiled with protoc 2.5.0
From source with checksum 71e2695531cb3360ab74598755d036
This command was run using /usr/local/hadoop-2.7.6/share/hadoop/common/hadoop-common-2.7.6.jar
```

## 4. Slave节点基本配置

按照3.1~3.3节的步骤，在Slave节点上配置hadoop用户、安装SSH、安装JDK环境。

## 5. 集群配置

### 5.1 网络配置

为了便于区分，可以修改各个节点的主机名（在终端标题、命令行中可以看到主机名，以便区分）。执行`sudo vim /etc/sysconfig/network`修改主机名HOSTNAME，这里将三个节点分别设置为moon111，moon112，moon113。

在**每个**节点上，执行如下命令，修改集群节点的主机名与IP地址映射关系：

```shell
sudo vim /etc/hosts
```

将原有的127.0.0.1和::1开头的两行注释掉，添加每个节点IP与主机的映射关系(注意这里要配置192开头的内网IP，使用ifconfig命令可以查询到)：

```shell
# 127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
# ::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

192.168.196.12 moon111
192.168.196.10 moon112
192.168.196.11 moon113
```

配置好后需要在各个节点上执行如下命令，测试是否相互 ping 得通，如果 ping 不通，后面就无法顺利配置成功：

```shell
[hadoop@moon111 etc]$ ping moon112 -c 3	# 只ping 3次，否则要按 Ctrl+c 中断
```

**注意：继续下一步配置前，请先完成所有节点的网络配置，修改过主机名的话需重启才能生效**。

### 5.2 SSH无密码登录节点

这个步骤是要让Master节点可以无密码SSH登录到各个Slave节点上。

首先生成Master节点的公钥，在Master节点的终端中执行

```shell
[hadoop@moon111 ~]$ cd ~/.ssh/	# 若该目录不存在，先执行一次ssh localhost
[hadoop@moon111 .ssh]$ ssh-keygen -t rsa	# 有提示的地方直接按回车就可以
[hadoop@moon111 .ssh]$ cat id_rsa.pub >> authorized_keys	# 加入授权
[hadoop@moon111 .ssh]$ chmod 600 ./authorized_keys    # 修改文件权限
```

完成后可执行 `ssh moon111` 验证一下（可能需要输入 yes，成功后执行 `exit` 返回原来的终端）。接着在 Master 节点将上公匙传输到 其他Slave节点：

```shell
[hadoop@moon111 .ssh]$ scp ~/.ssh/id_rsa.pub hadoop@moon112:/home/hadoop    
```

在**每个**Slave节点上，将ssh公钥加入授权：

```shell
[hadoop@moon112 ~]$ mkdir ~/.ssh	# 如果不存在该文件夹需先创建，若已存在则忽略
[hadoop@moon112 ~]$ cat ~/id_rsa.pub >> ~/.ssh/authorized_keys
[hadoop@moon112 ~]$ rm ~/id_rsa.pub	# 用完就可以删掉了
[hadoop@moon112 .ssh]$ chmod 700 ~/.ssh	# .ssh目录的权限必须是700
[hadoop@moon112 .ssh]$ chmod 600 ~/.ssh/authorized_keys	# .ssh/authorized_keys文件权限必须是600
```

在Master节点上执行如下命令进行校验，可无密码SHH登录至Slave节点：

```shell
[hadoop@moon111 .ssh]$ ssh moon112
[hadoop@moon112 ~]$ 
```

### 5.3 配置PATH变量

这一步是将hadoop安装目录加入PATH变量时，以便在任意目录下使用hadoop、hdfs等命令。在Master节点上执行`vim ~/.bashrc`，加入hadoop的安装路径，如下：

```
export PATH=$PATH:/usr/local/hadoop-2.7.6/bin:/usr/local/hadoop-2.7.6/sbin
```

保存后执行`source ~/.bashrc`使配置生效。

### 5.4 配置集群环境

在Master节点中，进入到Hadoop的配置目录`/usr/local/hadoop-2.7.6/etc/hadoop`，需要修改的文件包括：slaves、core-site.xml、hdfs-site.xml、mapred-site.xml、yarn-site.xml，具体配置如下。

**slaves**

删除localhost，增加两个Slave节点：

moon112

moon113

**core-site.xml**

```xml
<configuration>
        <property>
                <name>fs.defaultFS</name>
          		<!-- 修改为Master节点主机名 -->
                <value>hdfs://moon111:9000</value>
        </property>
        <property>
                <name>io.file.buffer.size</name>
                <value>131072</value>
        </property>
        <property>
                <name>hadoop.tmp.dir</name>
          		<!-- 可自定义目录 -->
                <value>file:/home/hadoop/tmp</value>
                <description>Abase for other temporary directories.</description>
        </property>
        <property>
                <name>hadoop.proxyuser.root.hosts</name>
                <value>*</value>
        </property>
        <property>
                <name>hadoop.proxyuser.root.groups</name>
                <value>*</value>
        </property>
</configuration>
```

**hdfs-site.xml**

```xml
<configuration>
        <property>
                <name>dfs.replication</name>
                <value>3</value>
        </property>
        <property>
                <name>dfs.namenode.name.dir</name>
                <value>file:///home/hadoop/dfs/name</value>
        </property>
        <property>
                <name>dfs.datanode.data.dir</name>
                <value>file:///home/hadoop/dfs/data</value>
        </property>
        <property>
                <name>dfs.namenode.secondary.http-address</name>
                <value>moon111:9001</value>
        </property>
        <property>
                <name>dfs.webhdfs.enabled</name>
                <value>true</value>
        </property>
        <property>
                <name>dfs.permissions</name>
                <value>false</value>
        </property>
</configuration>

```

**mapred-site.xml**

**注：**需要先重命名，默认文件名为 mapred-site.xml.template

```xml
<configuration>
         <property>
                 <name>mapreduce.framework.name</name>
                 <value>yarn</value>
         </property>
         <property>
           		 <name>mapreduce.jobhistory.address</name>                 					<value>moon111:10020</value>
         </property>         
  		 <property>
           		 <name>mapreduce.jobhistory.webapp.address</name>                 			 <value>moon111:19888</value>         
  		 </property>
</configuration>
```

**yarn-site.xml**

```xml
<configuration>
        <property>
                <name>yarn.nodemanager.aux-services</name>
                <value>mapreduce_shuffle</value>
        </property>
        <property>
                <name>yarn.nodemanager.aux-services.mapreduce.shuffle.class</name>
                <value>org.apache.hadoop.mapred.ShuffleHandler</value>
        </property>
        <property>
                <name>yarn.resourcemanager.address</name>
                <value>moon111:8032</value>
        </property>
        <property>
                <name>yarn.resourcemanager.scheduler.address</name>
                <value>moon111:8030</value>
        </property>
        <property>
                <name>yarn.resourcemanager.resource-tracker.address</name>
                <value>moon111:8035</value>
        </property>
        <property>
                <name>yarn.resourcemanager.admin.address</name>
                <value>moon111:8033</value>
        </property>
        <property>
                <name>yarn.resourcemanager.webapp.address</name>
                <value>moon111:8088</value>
        </property>
</configuration>
```

另外，还需要修改该目录下的**hadoop-env.sh**和**yarn-env.sh**两个文件，将其中的JAVA_HOME改为实际的JDK安装路径：

```shell
export JAVA_HOME=/usr/java/jdk1.8.0_121
```

配置完成后，将Master节点上的hadoop目录复制到各个从节点上，执行命令如下：

```shell
[hadoop@moon111 hadoop]$ cd /usr/local
[hadoop@moon111 local]$ tar -zcf ~/hadoop.master.tar.gz ./hadoop-2.7.6	# 文件较大，建议先压缩再复制
[hadoop@moon111 local]$ cd ~
[hadoop@moon111 ~]$ scp ./hadoop.master.tar.gz moon112:/home/hadoop    
[hadoop@moon111 ~]$ scp ./hadoop.master.tar.gz moon113:/home/hadoop
```

在**每个**Slave节点上执行：

```shell
sudo tar -zxf ~/hadoop.master.tar.gz -C /usr/local
sudo chown -R hadoop /usr/local/hadoop
```

启动hadoop前，需要关闭**每个**节点的防火墙，可以通过如下命令关闭防火墙：

```shell
sudo service iptables stop
```

另外，首次启动需要先在 Master 节点执行 NameNode 的格式化：

```shell
[hadoop@moon111 ~]$ hdfs namenode -format
```

在Master节点上启动hadoop：

```shell
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver
```

启动后，通过jps命令可以查看各个节点所启动的进程。正确的话，Master节点可以看到NameNode、ResourceManager、SecondrryNameNode、JobHistoryServer 进程，如下：

```shell
[hadoop@moon111 hadoop]$ jps
1978 SecondaryNameNode
2490 Jps
2443 JobHistoryServer
2155 ResourceManager
1774 NameNode
```

Slave节点可以看到 DataNode 和 NodeManager 进程，如下所示：

```shell
[hadoop@moon112 hadoop]$ jps
1511 Jps
1290 DataNode
1390 NodeManager
```

出现以上进程提示即表明hadoop集群环境配置成功。

另外还需要在 Master 节点上通过命令 `hdfs dfsadmin -report` 查看 DataNode 是否正常启动，如果 Live datanodes 不为 0 ，则说明集群启动成功。如果datanodes为0，查看启动日志排查原因。

```shell
[hadoop@moon111 ~]$ hdfs dfsadmin -report
Configured Capacity: 169094021120 (157.48 GB)
Present Capacity: 131891498491 (122.83 GB)
DFS Remaining: 131890532352 (122.83 GB)
DFS Used: 966139 (943.50 KB)
DFS Used%: 0.00%
Under replicated blocks: 16
Blocks with corrupt replicas: 0
Missing blocks: 0
Missing blocks (with replication factor 1): 0

-------------------------------------------------
Live datanodes (2):

Name: 192.168.196.10:50010 (moon112)
Hostname: moon112
...
```

关闭 Hadoop 集群也是在 Master 节点上执行的，命令如下：

```shell
stop-yarn.sh
stop-dfs.sh
mr-jobhistory-daemon.sh stop historyserver
```

## 参考资料

- Hadoop2.7.3在CentOS 6.5中的集群搭建    <https://blog.csdn.net/chenshijie2011/article/details/75670086>

- Hadoop安装教程_伪分布式配置_CentOS6.4/Hadoop2.6.0    <http://dblab.xmu.edu.cn/blog/install-hadoop-in-centos/>

- Hadoop集群安装配置教程_Hadoop2.6.0_Ubuntu/CentOS    <http://dblab.xmu.edu.cn/blog/install-hadoop-cluster/>