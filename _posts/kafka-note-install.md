---
title: "kafka学习笔记1:测试环境搭建"
date: 2017/09/24 15:00:00
---
最近因为架构中引入了kafka,一些之前在代码中通过RPC调用强耦合但是适合异步处理的内容可以用kafka重构一下.  
考虑从头学一下kafka了解其特性和使用场景。  
  
# 环境选择  

首先是测试环境的搭建,平时使用的是win,但kafka以及zk在win上会存在一些bug(例如 https://issues.apache.org/jira/browse/KAFKA-1194),最好还是在linux平台上搭建.
虚拟机是一个不错的选择但开销比较大,日常使用的笔记本8G内存开启虚拟机不是很方便,`bash on windows`是个不错的选择.
(另在`Store`中可以下载`ubuntu`等子系统,现在只限于insider)  
![](/images/1244488-20170924141841165-1167762603.png)  
在组件中开启后重启电脑更新.

# 环境搭建  

重启后,在命令行或者powershell中输入bash进入环境.  
![](/images/1244488-20170924142156571-346165170.png)  
但这里的环境和外部的windows不互通,windwos装了java也无法在bash中使用.  
需要先安装一些额外的组件.  

## java安装  
![](/images/1244488-20170924142211259-23564339.png)  
输入后选择一个即可(这里选择了openjdk-8-jre-headless)

## zookeeper  
kafka依赖zk  
简单的关系如下(图片来源kafka权威指南):  
![](/images/1244488-20170924142622790-2144780732.png)  
所以使用kafka之前需要配置好zk.  

	cd ~
	wget https://mirrors.tuna.tsinghua.edu.cn/apache/zookeeper/zookeeper-3.4.6/zookeeper-3.4.6.tar.gz
	tar -zxf zookeeper-3.4.6.tar.gz
	mv zookeeper-3.4.6 /usr/local/zookeeper
	mkdir -p /var/lib/zookeeper
	vim /usr/local/zookeeper/conf/zoo.cfg
	/usr/local/zookeeper/bin/zkServer.sh start  

zoo.cfg的内容和zoo_sample.cfg大致相同,将dataDir换为`/var/lib/zookeeper`即可.  
内容如下:    

    tickTime=2000
    dataDir=/var/lib/zookeeper
    clientPort=2181
启动后zk在2181端口运行  

## kafka  
kafka安装过程类似.  
这里学习使用0.9.0.1版本,去年2月发布.  
和所看的书保持一致避免出现不兼容问题导致书中例子无法运行阻碍进度(新版本的功能之后再额外学习).  

	wget http://mirror.bit.edu.cn/apache/kafka/0.9.0.1/kafka_2.11-0.9.0.1.tgz
	tar —zxf kafka 2.11-0.9.0.1.tgz
	mv kafka 2.11-0.9.0.1 /usr/local/kafka
	mkdir /tmp/kafka—logs
	/usr/local/kafka/bin/kafka—server—start.sh -daemon /usr/local/kafka/config/server.properties

创建topic并验证: 

	# /usr/local/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181
	--replication-factor 1 --partitions 1 --topic test
	Created topic "test".
	# /usr/local/kafka/bin/kafka-topics.sh --zookeeper localhost:2181
	--describe --topic test
	Topic:test    PartitionCount:1    ReplicationFactor:1    Configs:
		Topic: test    Partition: 0    Leader: 0    Replicas: 0    Isr: 0

同时zk的/brokers/topics下会多出一个test.  

至此整个安装过程就结束了.  

上面的`/usr/local/kafka/config/server.properties`是broker的配置,其中一些配置的含义如下:  

* broker.id  
都必须要有一个id 默认是0  
在一个集群里必须唯一

* port  
监听的端口 默认是9092  
指定1024以下的端口要用root跑 但不建议

* zookeeper.connent  
连接zk的地址 默认是localhost的2181  
格式是hostname:port/path  
默认path不写是根  

* log.dirs  
log块的存储位置  
可以用，分隔存多个  
一个分区的日志会存在一个路径下  

* num.recovery.threads.per.data.dir  
用来启动和结束时做日志恢复相关  
和dirs的数量相关 如果是3 dirs数量是2 那会有6个线程

* num.partitions  
一个新的topic会有多少个分区  
分区数只能比这个数字大 不能比这个小
