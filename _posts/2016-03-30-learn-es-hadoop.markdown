---
layout: post
title:  "Elasticsearch-hadoop学习笔记"
date:   2016-03-30 18:02:00 +0800
category: 学习笔记
tagline: "Supporting tagline"
tags: [elasticsearch, es, hadoop]
---

---

# 1. WWH

##### What?
elasticsearch与hadoop的结合
官方给出了两个概念:

	es-yarn
	es-hadoop

##### Why?
为了能让公司的LEK框架和青云的Hadoop, Spark结合

##### How?
没深入研究前的几个疑问:

1. **es的数据存储是否可以是hdfs?**
2. **hadoop中各组件怎么可以通过API操作es**
3. **spark能否读取es中的数据作为rdd, 能否将计算结果写入es**
4. **es-yarn和es-hadoop各自是什么?**
5. **es是独立安装还是集成到hadoop中? 如果是独立安装, 如何关联**


# 2. 开始折腾

### 2.1 盲人摸象

现在对上述疑问还是云里雾里, 所以只能摸着石头过河

首先在青云上创建了一个Hadoop集群和一个带有hadoop/spark客户端的主机, 我们称作试验机(TM)

基本上是按照es的官方文档来做
<https://www.elastic.co/guide/en/elasticsearch/hadoop/current/ey-usage.html#yarn-es-download>

以root用户登录到TM上, 下载es-hadoop

**wget http://download.elastic.co/hadoop/elasticsearch-hadoop-2.2.0.zip**

解压后进入elasticsearch-hadoop-2.2.0/dist目录

尝试下hadoop jar elasticsearch-yarn-2.2.0.jar

```
No command specified
Usage:
     -download-es  : Downloads Elasticsearch.zip
     -install      : Installs/Provisions Elasticsearch-YARN into HDFS
     -install-es   : Installs/Provisions Elasticsearch into HDFS
     -start        : Starts provisioned Elasticsearch in YARN
     -status       : Reports status of Elasticsearch in YARN
     -stop         : Stops Elasticsearch in YARN
     -help         : Prints this help

Configuration options can be specified _after_ each command; see the documentation for more information.

```

提示需要输入具体命令

输入**hadoop jar elasticsearch-yarn-2.2.0.jar -download-es**

显示正在下载(不是都下载过啦, 为毛又要下载)

```
Downloading Elasticsearch 2.2.0
Downloading .........................................DONE
```

下载完毕发现当前目录下多了个downloads目录, 进去查看, 发现elasticsearch-2.2.0.zip!

现在明白了, 原来下载的是es的正宗安装包, 之前下载过的elasticsearch-hadoop-2.2.0.zip只是关联es和hadoop的插件而已, 实际上这一步可以替换为, 自己创建一个downloads目录, 然后自己去官网下载安装包放进去.

接下来文档中提示的是Provision Elasticsearch into HDFS, 翻译一下是把es提供给HDFS, 那么是做什么呢, 我这里猜想是通过HDFS将elasticsearch-2.2.0.zip分发给所有的HADOOP的节点!以达到通过在各个节点上安装并由yarn来启动分布式的es的目的!

这里就有一个问题, **如果自己需要为es安装插件或者修改配置怎么办?**, 答案很可能是把安装包解压开之后, 将插件下载到指定的目录中, 修改配置文件, 再压缩为zip, 目前先不管这个问题, 安装成功之后再说

现在终于有点头绪了, **es-yarn是一个插件, 作用就是可以把es安装到hadoop集群中, 并由yarn来管理启动**, 接下来是真正的安装了

### 2.2 在hadoop集群上安装elasticsearch


返回上级目录执行:
hadoop jar elasticsearch-yarn-2.2.0.jar -install-es

果断报错

```
Abnormal execution:Cannot upload /home/test/elasticsearch-hadoop-2.2.0/dist/./downloads/elasticsearch-2.2.0.zip in HDFS

...

Caused by: java.io.IOException: Incomplete HDFS URI, no host: hdfs:///

...
```
JAVA抛出的异常信息中最有价值的就是上面的几句话, 猜测是**找不到hdfs**

这个问题官方文档中没有给出, 为什么会出现? 是不是因为官方文档中默认是在hdfs的主节点上安装的, 也就是说这个安装命令会将zip包传到hdfs://127.0.0.1上?

尝试自己添加hdfs的路径

vim /etc/hosts, 按照青云的用户指南来修改下<https://docs.qingcloud.com/guide/hadoop.html>

```
192.168.107.20    localhost
192.168.107.20    i-tp5n8o28
192.168.107.55    hdpn-mmqm9blk-hdfs-master
192.168.107.66    hdpn-on161rse-yarn-master
192.168.107.77    hdpn-dflptij8-slave
192.168.107.88    hdpn-dggp37ij-slave
192.168.107.99    hdpn-kjtvfg34-slave
```

执行: hadoop jar elasticsearch-yarn-2.2.0.jar -install-es hdfs://hdpn-hdaiiv16-hdfs-master:9000

然并卵, 并没有一个参数可以指定hdfs的路径

其实, 很SB的原因, es官方文档上还是给出了的, 一句不起眼的话:This command uploads the elasticsearch-<version>.zip (that we just downloaded) to HDFS (based on the Hadoop configuration detected in the classpath) under /apps/elasticsearch folder.

原因很简单, 我们创建了TM之后都还没有配置过hadoop中的core-site.xml!!!

按照青云的文档配置即可:

vim /usr/local/hadoop/etc/hadoop/core-site.xml

```
#将以下内容写入 core-site.xml 文件来配置 HDFS 的地址
<configuration>
        <property>
            <name>fs.defaultFS</name>
            <value>hdfs://hdpn-mmqm9blk-hdfs-master:9000</value>
        </property>
</configuration>
```

同时把yarn-site.xml, mapred-site.xml一并修改

vim  /usr/local/hadoop/etc/hadoop/yarn-site.xml

```
<configuration>
  <property>
      <name>yarn.nodemanager.aux-services</name>
      <value>mapreduce_shuffle</value>
  </property>
  <property>
      <name>yarn.nodemanager.aux-services.mapreduce_shuffle.class</name>
      <value>org.apache.hadoop.mapred.ShuffleHandler</value>
  </property>
  <property>
      <name>yarn.resourcemanager.address</name>
      <value>hdpn-on161rse-yarn-master:8032</value>
  </property>
  <property>
      <name>yarn.resourcemanager.scheduler.address</name>
      <value>hdpn-on161rse-yarn-master:8030</value>
  </property>
  <!--property>
      <name>yarn.resourcemanager.resource-tracker.address</name>
      <value>hdpn-on161rse-yarn-master:8031</value>
  </property>
  <property>
      <name>yarn.resourcemanager.admin.address</name>
      <value>hdpn-on161rse-yarn-master:8033</value>
  </property-->
  <property>
      <name>yarn.resourcemanager.webapp.address</name>
      <value>hdpn-on161rse-yarn-master:8088</value>
  </property>
</configuration>
```

vim /usr/local/hadoop/etc/hadoop/mapred-site.xml

```
<configuration>
  <property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
  </property>
</configuration>
```

再来:**hadoop jar elasticsearch-yarn-2.2.0.jar -install-es**

```
Uploaded /home/test/elasticsearch-hadoop-2.2.0/dist/./downloads/elasticsearch-2.2.0.zip to HDFS at hdfs://hdpn-hdaiiv16-hdfs-master:9000/apps/elasticsearch/elasticsearch-2.2.0.zip
```

成功!

继续按照官方文档走, Provision Elasticsearch-YARN into HDFS, 意思是讲es-yarn传到hdfs中, 根据结果来看实际上就是es-yarn这个插件本身装进去

**hadoop jar elasticsearch-yarn-2.2.0.jar -install**

```
Uploaded /home/test/elasticsearch-hadoop-2.2.0/dist/elasticsearch-yarn-2.2.0.jar to HDFS at hdfs://hdpn-hdaiiv16-hdfs-master:9000/apps/elasticsearch/elasticsearch-yarn-2.2.0.jar
```
实际上到现在, 我们完成了安装过程

### 2.3 在hadoop集群上启动elasticsearch

启动一个3节点的es
**hadoop jar elasticsearch-yarn-2.2.0.jar -start containers=3**

```
Launched a 3 nodes Elasticsearch-YARN cluster [application_1458812461990_0002@http://hdpn-7xypk7xi-yarn-master:8088/proxy/application_1458812461990_0002/] at Thu Mar 24 19:10:28 CST 2016
```

可以在yarn的管理UI上查看:

http:/192.168.10.12:8088/cluster

关闭:

**hadoop jar elasticsearch-yarn-2.2.0.jar -stop**

**但是....在yarn的管理UI上查看, es这个application居然是TM失败的!!**

...为啥, 为啥..

**好吧, 我猜是青云的hadoop集群中, 对yarn做了限制, 并不允许我们自己去安装自定义的applicaiton??? 这个问题 只能找青云探讨了...**

这个问题我在查资料的过程中找到了两种可能的解决方式:
<https://discuss.elastic.co/t/unable-to-start-elasticsearch-yarn-container-is-running-beyond-virtual-memory-limits/26102>

```
方式一:

<property>

   <name>yarn.nodemanager.vmem-check-enabled</name>

   <value>false</value>

</property>

方式二:

修改下面的属性, 从默认的2.1提高到4

<property>

   <name>yarn.nodemanager.vmem-pmem-ratio</name>

   <value>2.1</value>

</property>


```

但是让青云的工作人员调整过参数之后, 虽然报错信息改变了, 但是还是失败. 后来经过排查发现是因为用root用户执行的启动命令, 那么果断切到ubuntu用户, sudo -i -u ubuntu, 但是还是报错.


# 3 新的一步

2.3中的出现的问题, 我在我们在阿里云上搭建的hadoop集群(hdp2.3)上进行上述安装, 已经成功.

这里先暂且不管青云集群出现的问题, 继续向下进行.

到这一步, 又查看了es-hadoop的文档, 我们前面提到几个疑问:

```
1. **es的数据存储是否可以是hdfs?**
2. **hadoop中各组件怎么可以通过API操作es**
3. **spark能否读取es中的数据作为rdd, 能否将计算结果写入es**
4. **es-yarn和es-hadoop各自是什么?**
5. **es是独立安装还是集成到hadoop中? 如果是独立安装, 如何关联**
```

其中的4和5已经有了答案:

**es-yarn是一个hadoop的插件, 具体来说就是elasticsearch-yarn-2.2.0.jar, 我们可以通过它来在yarn上部署管理elasticsearch, 而es-hadoop是一个hadoop生态圈和elasticsearch之间的桥梁, 可以让hadoop各种数据分析工具从es中读写数据, 所以问题2和3也有了答案**

**es是通过安装到hadoop中去的, 通过yarn管理**

而问题1在查资料和读文档的过程中也有了印证:
<http://bigbo.github.io/pages/2015/02/28/elasticsearch_hadoop/>

1. es的数据存储可以是hdfs, 但是是通过一个官方已经放弃维护的elasticsearch-hdfs插件实现, 而使用者纷纷表示性能太差
2. 官方提出了一个gateway的概念, 可以将数据**备份/快照**到hdfs中, 用于es恢复数据, 这个功能对我们没什么卵用, 除非我们能直接读取hdfs中的快照数据, 但是预计难度会比较大.


spike进行到这里, 我们已经有了比较清晰的概念:

1. 我们将会把es部署到hadoop上, 用yarn管理
2. es的数据存储仍使用本地存储
3. 我们将通过es-hadoop, 使用spark读写es, 以提高我们对日志的分析性能

# 4 elasticsearch

对于es本身, 现在有3个问题:

1. 如何修改配置
2. 通过yarn部署的es集群(container>1)是怎么样实现的
3. 如何安装插件

### 4.1 配置修改
因为es是通过把安装包上传到hdfs中, 由yarn启动的, 所以配置只能在上传之前做修改.


配置详解: <http://rockelixir.iteye.com/blog/1883373>

进入elasticsearch/dist/downloads目录, 里面的就是刚才通过-download-es下载的zip包, 解压

unzip elasticsearch-hadoop-2.2.0.zip

vim elasticsearch-hadoop-2.2.0/config/elasticsearch.yml

修改如下配置

##### 集群配置
```
#---------------------------------- Cluster -----------------------------------
#
# Use a descriptive name for your cluster:
#
cluster.name: tataufo
#

```
es对集群的定义是: cluster.name相同的es各实例, 如果能够通过unicast方式发现彼此, 则共同组成一个集群

可以通过设置index.number_of_shards和index.number_of_replicas来设置分片数和副本数, 这里不做设置, **es-yarn应该会自己管理?**

##### 网络配置

```
#允许访问的主机IP, 0.0.0.0代表任意主机
network.host: 0.0.0.0
http.port: 9200
```

##### unicast discovery(集群节点发现配置)

```
discovery.zen.ping.unicast.hosts: ["x.x.x.x", "x.x.x.x"]
```



默认是127.0.0.1, 即只将本机的cluster.name相同的es作为集群中的节点


##### 配置生效

重新打包上传安装

rm -rf elasticsearch-hadoop-2.2.0.zip

zip -r elasticsearch-hadoop-2.2.0.zip elasticsearch-hadoop-2.2.0

hadoop jar elasticsearch-yarn-2.2.0.jar -install-es

hadoop jar elasticsearch-yarn-2.2.0.jar -install

hadoop jar elasticsearch-yarn-2.2.0.jar -start containers=x

这里的x即es集群将启动的节点数

### 4.2 安装插件

同样需要解压安装包, 安装插件后重新上传安装启动

unzip elasticsearch-hadoop-2.2.0.zip

cd elasticsearch-hadoop-2.2.0

head插件: bin/plugin install mobz/elasticsearch-head

marvel-agent插件:

	bin/plugin install license
	bin/plugin install marvel-agent

	marvel插件需要在kibana中安装marvel server
	bin/kibana plugin --install elasticsearch/marvel/latest

打包后重新上传安装并启动

head界面:
http://xx:9200/_plugin/head/

marvel界面需要在启动kibana后查看:
http://xx:5601/app/marvel
