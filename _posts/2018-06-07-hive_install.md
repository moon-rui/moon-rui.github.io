---
layout: article
title: Hive安装配置
key: 2018-06-07-hive_install
tags: Hive
---

## 1. 环境说明

三台CentOS 6.5 64位环境服务器，并已安装好Hadoop集群环境。

Hive使用2.3.3版本，官网下载地址http://archive.apache.org/dist/hive/

## 2. 安装

下载完成后解压安装包：

**这里需要注意的是，Hive不需要像HBase、Zookeeper那样集群安装，只需安装在Master节点即可。**

```shell
tar -xzvf apache-hive-2.3.3-bin.tar.gz
mv apache-hive-2.3.3-bin hive-2.3.3	# 重命名文件夹
```

## 3. 配置

为了方便使用，我们把hive命令加入到环境变量中去，需要使用以下命令编辑.bashrc文件：

```shell
vim ~/.bashrc       # 设置环境变量
```

进入编辑状态以后，需要添加如下几行：

```shell
export HIVE_HOME=/usr/local/hive
export PATH=$PATH:$HIVE_HOME/bin
```

完成上述操作后，需要运行以下命令让配置生效：

```shell
source ~/.bashrc
```

进入hive的conf目录，将hive-default.xml.template 文件复制为hive-site.xml

```shell
cd hive-2.3.3/conf/
cp hive-default.xml.template hive-site.xml
```

**注意：**在安装Hive时，默认情况下，元数据存储在Derby数据库中。Derby是一个完全用Java编写的数据库，所以可以跨平台，但需要在JVM中运行。因为多用户和系统可能需要并发访问元数据存储，所以默认的内置数据库并不适用于生产环境。任何一个适用于JDBC进行连接的数据库都可用作元数据库存储，这里我们把MySQL作为存储元数据的数据库。 

因此在做进一步配置前，需要安装好MySQL环境，并创建相应的用户，命令如下：

```sql
CREATE USER 'hive' IDENTIFIED BY 'hive';   # 创建hive用户
create database hive;                  # 创建hive数据库
grant all on hive.* to hive@'%' identified by 'hive';
grant all on hive.* to hive@'localhost' identified by 'hive'; 
flush privileges;
exit                   #退出mysql
    
mysql -u hive -p hive        #验证hive用户
show databases;    
```

**配置hive-site.xml**

其中数据库连接信息根据实际安装的MySQL环境进行配置。

```xml
<configuration>
    <property>
        <name>hive.exec.scratchdir</name>
        <value>/tmp/hive-${user.name}</value>
        <description>HDFS root scratch dir for Hive jobs which gets created with write all (733) permission. For each connecting user, an HDFS scratch dir: ${hive.exec.scratchdir}/&lt;username&gt; is created, with ${hive.scratch.dir.permission}.</description>
    </property>
    <property>
        <name>hive.exec.local.scratchdir</name>
        <value>/tmp/${user.name}</value>
    <description>Local scratch space for Hive jobs</description>
    </property>
    <property>
        <name>hive.downloaded.resources.dir</name>
        <value>/tmp/hive/resources</value>
        <description>Temporary local directory for added resources in the remote file system.</description>
    </property>
    <property>
        <name>hive.querylog.location</name>
        <value>/tmp/${user.name}</value>
        <description>Location of Hive run time structured log file</description>
    </property>
    <property>
        <name>hive.server2.logging.operation.log.location</name>
        <value>/tmp/${user.name}/operation_logs</value>
        <description>Top level directory where operation logs are stored if logging functionality is enabled</description>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <value>jdbc:mysql://localhost:3306/hive?createDatabaseIfNotExist=true</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>hive</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>hive</value>
    </property>
</configuration>
```

**配置hive-env.sh**

```shell
cp hive-env.sh.template hive-env.sh	# 从模板文件中复制
```

配置内容如下：

```shell
export JAVA_HOME=/usr/java/jdk1.8.0_121
export HIVE_HOME=/home/hadoop/hive-2.1.1
export HADOOP_HOME=/home/hadoop/hadoop-2.7.3
export PATH=$JAVA_HOME/bin:$PATH:$HIVE_HOME/bin:$HADOOP_HOME/bin
export CLASSPATH=.:$JAVA_HOME/lib/dt.jar:$JAVA_HOME/lib/tools.jar:$HIVE_HOME/lib:/home/hadoop/hbase-1.2.4/lib
export HIVE_CONF_DIR=$HIVE_HOME/conf
```

修改完hive-site.xml文件后，需要把JDBC驱动放置在hive的lib目录下（JDBC驱动程序mysql-connector-java-x.x.x-bin.jar文件的下载地址为<http://www.mysql.com/downloads/connector/j/> ，建议使用5.1.XX版本）。

## 4. 启动

启动Hive前，需要先启动hadoop，再启动HBase，如果不需要通过hive使用HBase，可以不启动HBase。

从 Hive 2.1 版本开始, 我们需要先运行 schematool 命令来执行初始化操作：

```shell
schematool -dbType mysql -initSchema
```

启动hive时有几种不同的命令：

- `hive`  直接启动hive，进入到hive的命令行下，在命令行模式下进行hive操作
- `hive -hiveconf hive.root.logger=DEBUG,console`  带日志启动hive，还是在命令行模式下使用hive，但是会输出详细的信息，调试错误时非常有用
- `hive –service hwi`  带web接口启动hive，此时可以在浏览器打开hive的管理页面，前提是部署了hive-hwi-2.1.1.war。
- `hive –service hiveserver`  如果要在java代码里调用hive，必须这样启动

用第二种方式启动，进入命令行

> hive -hiveconf hive.root.logger=DEBUG,console

启动成功后用jps查看，会看到一个RunJar进程。

停止hive可在shell下直接输入quit命令

> quit;

注意hive的命令结尾要加分号，和hbase不同。

这个时候再用jps命令就看不到RunJar进程了。

## 5. 创建Hive表

### 5.1 外部表

可以在hbase中先创建一个表，然后在hive里创建一个外部表来关联hbase中的表,创建外部表的时候hive只保存表的元数据。

进入到hbase的shell下执行：

> create ‘member’,’id’,’address’
>
> put ‘member’,’r1’,’id:address’,’hangzhou’

**注意：**hive关联hbase的表需要把hbase/lib和hadoop/share/hadoop/common/lib下的一些jar包拷贝到hive/lib目录下，包括Hbase下的`hbase-client-2.0.0.jar`，`hbase-server-2.0.0.jar`，`hbase-common-2.0.0.jar`，`hbase-protocol-2.0.0.jar`, `htrace-core-3.2.0-incubating.jar`。Hadoop下的 /home/hadoop/hadoop-2.7.6/share/hadoop/common 目录下的`hadoop-common-2.7.6.jar`

复制完jar包后要重启hive。进入hive的shell，关联hbase中的member表：

```sql
CREATE EXTERNAL TABLE hbase_member(key string, value string) STORED BY 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' WITH SERDEPROPERTIES("hbase.columns.mapping"="id:address") TBLPROPERTIES("hbase.table.name"="member");
```

完成后可查询到在Hbase中插入的记录：

```shell
hive> select * from hbase_member;
OK
r1	hangzhou
Time taken: 1.683 seconds, Fetched: 1 row(s)
```

### 5.2 内部表

hbase中没有创建表，直接在hive中创建内部表，间接在hbase中创建出表，在hive中执行命令：

```sql
create table hbase_table_1(key int, value string) stored by 'org.apache.hadoop.hive.hbase.HBaseStorageHandler' with serdeproperties("hbase.columns.mapping"=":key,cf1:val") tblproperties("hbase.table.name"="xyz");
```

进入Hbase的shell界面，输入list命令，可以看到在hive中建立的xyz表：

```shell
hbase(main):001:0> list
TABLE                                                                                                        
member                                                                                                    
table1                                                                                                        
xyz                                                                                                           
3 row(s)
Took 0.5954 seconds                                                                                      
=> ["member", "table1", "xyz"]
```

## 参考资料

- Hive-2.1.1的安装 https://blog.csdn.net/chenshijie2011/article/details/76147357

- linux中hive安装和部署详解 https://blog.csdn.net/a123demi/article/details/72742279

