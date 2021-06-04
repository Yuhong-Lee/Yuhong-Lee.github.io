---
layout: post
title: "Big Data -- 离线数据仓库项目概览"
description: "对于离线数据仓库的总结"
categories: [Big-Data]
tags: [Big-data, Data-Warehouse]
author: Lee
redirect_from:
  - /2021/06/04
---
# 离线数据仓库
## 数据仓库搭建准备
### 数据仓库的分层
ods --- optional data source 原始数据层用来对原始数据进行备份

dwd -----  数据仓库建模所在的层，维度建模在此层完成

dws ----- 对数据仓库的数据进行初步汇总，以主题对象为主、

dwt ------- 对数据仓库中的数据进行进一步的汇总，以主题按周期汇总

ads ------ 数据仓库的应用层，根据实际业务需求进行创建，定义，设计

### 范式

数据的三范式：一套标准的规范用于定义并规范表的创建

数据的冗余会造成：磁盘空间的浪费，数据一致性的问题难以保持

第一范式：属性不可切割

第二范式：不存在部分函数依赖

第三范式：不存在传递函数依赖，不能存在非主键字段传递依赖主键的现象
关系建模与维度建模

### 关系建模：

OLTP：online transaction processing，关系建模的典型应用，其主要是处理传统的关系型数据库中的数据，以基本，日常的数据为主。面向的是用户，可以随机低延时的写入用户的输入，可以表达数据的最新数据状态
表多，数据冗余较少
在进行的查询的时候需要较多的join才能实现，用时间换取了空间
维度模型：

OLAP：online analytical processing, 维度建模的典型应用，其主要是对于大量的记录数据进行汇总与分析，其中数据以批量写入为特性，供内部的数据分析师使用，数据的表征是随着时间变化的历史状态。
表少，数据冗余较高，相较于关系模型更容易理解，用空间换时间

更加适合用做聚合分析用途

### 维度表与事实表：
#### 维度表：	

从如何描述业务的角度考虑：维度表达的是对事实的信息的描述
从数据分析的角度考虑：维度表表达的是分析事物的角度
行数较少，范围较宽
内容相对固定 
#### 事实表：

事实表中的每行数据代表的是一次业务的事件。“事实”这个业务术语表达的是表示的是业务事件的度量值（个数，金额，件数等）。
事实表较大，经常发生变化，内容相对较窄
事物型事实表
一旦提交就不会更改，保留所有数据，更新方式为增量

    增量同步
        周期型快照事实表
        不保留所有数据，只保留固定时间间隔的数据，我们更加关心在固定时间间隔之后所保留的数据
    全量同步
        累计型快照事实表
        用于追踪业务事实的变化
        新增及变化

## 数据仓库搭建

### ODS

保持数据的原始状态不做修改，通过sqoop，flume等工具将数据导入到hive中
使用压缩方式存储以节约空间
创建分区表防止后续操作全表扫描
### DWD

    在DWD层中需要进行维度建模，一般采用星型模型，呈现一种星座模型的状态
    维度建模一般步骤：
    选择业务过程------声明粒度-------确认维度--------确认事实	

    选择业务过程：在业务系统中挑选感兴趣的业务流程，一条业务对应一个事实表
    
    声明粒度：声明粒度的意义是明确一条数据所要表示的是什么，尽可能选择最小的粒度以满足不同的需求

    确认维度：维度的目的是用来描述业务的事实，主要表达“谁，何时，何处”等信息

    确认事实：“事实”是指对于业务中的具体度量值，例如下单次数，订单金额等等

### DWS 
    统计各个主题当天的行为，以维度为基础
### DWT
    与DWS相互对应，统计累计行为，同样是以维度为基础
--------------------------------------------------------------------------------
## 集群准备
hadoop--略过，参考hadoop篇

## hive配置：
这里需要配置hive的环境变量以使其运算引擎为spark即 hive on spark
其中spark部分与hive部分查看spark与hive篇

首先配置Hive On Spark
添加环境变量

    sudo vim /etc/profile.d/my_env.sh

    export SPARK_HOME=/opt/module/spark
    export PATH=$PATH:$SPARK_HOME/bin

使其生效

    source /etc/profile.d/my_env.s

配置spark运行环境

    mv /opt/module/spark/conf/spark-env.sh.template /opt/module/spark/conf/spark-env.sh
    vim /opt/module/spark/conf/spark-env.sh
添加
    
    export SPARK_DIST_CLASSPATH=$(hadoop classpath)

新建spark配置文件
    
    vim /opt/module/hive/conf/spark-defaults.conf
添加如下内容

    spark.master                               yarn
    spark.eventLog.enabled                   true
    spark.eventLog.dir                        hdfs://hadoop102:8020/spark-history
    spark.executor.memory                    1g
    spark.driver.memory                   1g

在HDFS创建如下路径
    
    hadoop fs -mkdir /spark-history

上传Spark依赖到HDFS
    
    hadoop fs -mkdir /spark-jars
    hadoop fs -put /opt/module/spark/jars/* /spark-jars

修改hive-site.xml


    <!--Spark依赖位置-->
    <property>
        <name>spark.yarn.jars</name>
        <value>hdfs://hadoop102:8020/spark-jars/*</value>
    </property>
     
    <!--Hive执行引擎-->
    <property>
        <name>hive.execution.engine</name>
        <value>spark</value>
    </property>
     
    <!--Hive和spark连接超时时间-->
    <property>
        <name>hive.spark.client.connect.timeout</name>
        <value>10000ms</value>
    </property>

--注意：hive.spark.client.connect.timeout的默认值是1000ms，如果执行hive的insert语句时，抛如下异常，可以调大该参数到10000m

Hive on Spark测试

建表-测试insert-检查log

在Yarn中增加队列


    <property>
        <name>yarn.scheduler.capacity.root.queues</name>
        <value>default,hive</value>
        <description>
          The queues at the this level (root is the root queue).
        </description>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.default.capacity</name>
    <value>50</value>
        <description>
          default队列的容量为50%
        </description>
    </property>
同时为新加队列添加必要属性：

    <property>
        <name>yarn.scheduler.capacity.root.hive.capacity</name>
    <value>50</value>
        <description>
          hive队列的容量为50%
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.user-limit-factor</name>
    <value>1</value>
        <description>
          一个用户最多能够获取该队列资源容量的比例
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.maximum-capacity</name>
    <value>80</value>
        <description>
          hive队列的最大容量
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.state</name>
        <value>RUNNING</value>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.acl_submit_applications</name>
    <value>*</value>
        <description>
          访问控制，控制谁可以将任务提交到该队列
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.acl_administer_queue</name>
    <value>*</value>
        <description>
          访问控制，控制谁可以管理(包括提交和取消)该队列的任务
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.acl_application_max_priority</name>
    <value>*</value>
    <description>
          访问控制，控制用户可以提交到该队列的任务的最大优先级
        </description>
    </property>
     
    <property>
        <name>yarn.scheduler.capacity.root.hive.maximum-application-lifetime</name>
    <value>-1</value>
        <description>
          hive队列中任务的最大生命时长
    </description>
    </property>
    <property>
        <name>yarn.scheduler.capacity.root.hive.default-application-lifetime</name>
    <value>-1</value>
        <description>
          hive队列中任务的默认生命时长
    </description>
    </property>
该配置需要分发

之后在hive中创建表
