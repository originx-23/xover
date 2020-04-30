---
title: "Hadoop详解"
date: 2020-03-20T17:41:16+08:00
draft: false
---
##  定义

### 狭义

HDFS:	分布式文件系统, 解决海量数据存储

YARN: 	作业调度和集群资源管理框架, 解决资源任务调度

MAPREDUCE: 分布式运算编程框架, 解决海量数据计算

### 广义

生态圈

## 发展

谷歌的三篇论文:

- 谷歌分布式文件系统(GFS)
- 谷歌版的MAPREDUCE系统
- bigtable

## 特性优点

扩容能力

成本低

高效率

可靠性

# 集群搭建

## 集群简介(物理上分离, 逻辑上在一起)

HDFS: NameNode, DataNode, SecondaryNameNode

YARN: ResourcManager, NodeManager

MapReduce: 分布式运算框架, 打包运行在HDFS集群上.

### 部署方式

Standalone mode			独立模式

Pseudo-Distributed mode	伪分布式

Cluster mode				集群模式

前两种都是在单机上部署.

### 服务器系统设置

#### 同步时间

```
yum install ntpdate
ntpdate cn.pool.ntp.org
```

#### 设置主机名

```
vi /etc/sysconfig/network
NETWORKING=yes
HOSTNAME=node-1
```

#### 配置IP,主机名映射

```
vi /etc/hosts
192.168.33.101	node-1
192.168.33.102	node-2
192.168.33.103	node-3
```

#### 配置ssh免密登录

```
# 生成ssh免登陆密钥
ssh-keygen -t rsa	(四个回车)
将生成的id_rsa.pub拷贝到要免密登录的目标机器上
ssh-copy-id node-02
```

#### 配置防火墙

```shell
# 查看防火墙
service iptables status
# 关闭防火墙
service iptables stop
# 查看防火墙开机启动状态
chkconfig iptables --list
# 关闭防火墙开机启动
chconfig iptables off
```

### 关于HDFS的格式化

首次启动需要格式化

格式化本质是进行一些文件系统的初始化操作,  创建一些需要的文件

格式化之后, 集群启动之后, 后续不再需要格式化

格式化操作在HDFS的主角色(namenode)所在机器上操作

```shell
hdfs namenode –format (hadoop namenode - format)
```

# HDFS基本

## 介绍

对海量索引文件进行存储

## 设计目标

故障检测和自动快速恢复

数据访问的高吞吐量

支持大文件

write-one-read-many

移动计算的代价比之移动数据的代价低

可移植性

## 重要特性

分布式文件系统

### master/slave架构

NameNode 是HDFS集群主节点, DataNode 是HDFS集群从节点.

### 分块存储

hadoop 2.x   块最小128M

### 名字空间

抽象一个目录树

### NameNode元数据管理

NameNode负责维护整个HDFS文件系统的目录树结构

### DataNode数据存储

存储多个副本, 默认是3( 一共三份)

定时向NameNode汇报自己持有的block信息

### 一次写入, 多次读出

不支持文件的修改, 延迟大, 网络开销大

## 基本操作

### Shell命令行客户端

```shell
hadoop fs <args>
# <args>: 将路径URI作为参数 scheme://authority/path
# 对于HDFS, scheme是hdfs; 对于本地FS, scheme是file; 如果未指定, 则使用配置中指定的默认方案 
hdfs def <args>老版本这个也可以
```

```shell
-ls -h	# 显示文件信息
-mkdir	# 指定路径创建目录
-put	# 上传文件	-p 保留访问和修改时间, 所有权和权限   -f 覆盖目的地(如果已经存在)
-get	# 将文件复制到本地文件系统
# -ignorecrc跳过对下载文件的CRC检验
-appendToFile	#追加到文件末尾
-chgrp	#修改组
-copyFromLocal	#put一样
-getmerger	#合并多个文件
-setrep	# 设置副本系数, 用于递归改变目录下所有文件的副本系数 -w 3 -R 路径
```

## 基本原理

### NameNode概述

- HDFS 的核心
- 也称为Master
- 仅存储元数据: 所有文件的目录树, 并跟踪整个集群中的文件
- 并不持久化存储每个文件中各个块在DataNode的位置信息, 每次启动重建
- NameNode所在机器通常配置大量内存条

### DataNode

- DataNode也成为Slave
- NameNode和DataNode保持通信
- 默认3秒发送一次心跳, 6小时汇报一次block

### 工作机制

Secondary NameNode协助NameNode进行元数据的备份

HDFS内部工作机制对客户端保持透明, 客户端请求访问HDFS都是通过向NameNode申请来进行

#### 写数据

默认三份, 本地一份, 同机架一份, 不同机架一份

## 应用开发

### 核心步骤

从HDFS提供的api中构造一个HDFS的访问客户端对象, 然后通过该客户端对象操作HDFS上的文件.

### 搭建开发环境

创建Maven工程一些jar包的引入

### Java相关操作

IOUtils( 输入流, 输出流) 

## MapReduce计算模型

### 思想

分而治之

Map负责分, 并行计算, 需要任务之间没有依赖关系;  Reduce负责合, 对map阶段的结果进行全局汇总

局部并行计算, 全局汇总

### 框架结构

一个完整的mapreduce 程序在分布式运行时有三类实例进程:

1. MRAppMaster: 负责整个程序的过程调度及状态协调
2. MapTask: 负责map阶段的整个数据处理流程
3. ReduceTask: 负责Reduce阶段的整个数据处理流程

![1553852410426](E:\文档\笔记\Hadoop.assets\1553852410426.png)

### 编程规范

1. 三个部分: Mapper, Reducer, Driver



### 案例

#### Mapper编写

```java
/*
JDK自带的类型,在序列化时候效率低下, hadoop 自己封装一套数据类型
long--->LongWritable
String--> Text
Interger--> Intwritable
null --> NullWritable
*/
```

#### Reducer编写

```java
/**
	reduce 接收所有来自map阶段处理的数据之后, 按照key的字典序进行排序
	按照key是否相同作为一组去调用reduce 方法
	本方法的key就是这一组相同kv对的共同key
	把这一组所有的v作为一个迭代器传入我们的reduce 方法
*/
public class WordCountReducer extends Reducer{
    protected void reduce(Text key, Iterable<IntWritable> value, Conut context) throws IOExpception,Interrupted Exception{
        int count = 0;
        for (IntWritable value: values){
                count += value.get();
            }
        context.write(key, new IntWritable(count));
    }
}
```

#### Driver编写

```java
/**
这个类就是mr程序运行时候的主类, 本类中组装了一些程序运行的时候所需信息
*/
public class WordCountDriver{
    public static void main(String[] args) throws Exception{
        // 通过Job来封装本次mr的相关信息
        Configuration conf = new Configuration();
        Job job = Job.getInstance(conf);
        
        //指定本次mr job jar包运行主类
        job.setJarByClass(WordCountDriver.class);
        
        //指定本次mr所用的mapper reducer类分别是
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);
        
        //指定本次mr mapper阶段的输出 kv 类型
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        
        //指定本次mr最终输出的kv 类型
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        
        //指定本次mr 输入的数据路径 和最终输出结果存放在什么位置
        FileInputFormat.setInputPaths(job, "/wordcount/input");
        FileOuntputFormat.setOutputPath(job, new Path("wordcount/output"));
        
        //job.submit
        //提交, 并打印执行情况
        boolean b = job.waitForCompletion(true);
        System.exit(b?0:1);
    }
}
```

