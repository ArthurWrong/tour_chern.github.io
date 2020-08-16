---
title: kylin简单教程
---

# 一、kylin概述
kylin提供的思路：预计算

## 多维立方体分析

- 维度

    观察数据的角度

    离散的值

- 度量
  
  聚合计算的结果

  连续的值

- Cube理论
  
  N个维度进行组合，一共有 $2^N$ 个组合

  每个组合做聚合运算，结果保存为一个物化视图（Cuboid）

  所有维度组合的Cuboid作为一个真题，即为一个Cube

  一个Cube就是许多按维度聚合聚合的物化视图的集合

## 技术架构

可分为在线和离线两个部分

![概览](http://kylin.apache.org/assets/images/kylin_diagram.png)

sql语句是基于数据源的关系模型编写的，最终会被转译成基于Cube的物理执行计划

## kylin三大模块：

1. 数据源

2. 构建引擎

3. 存储引擎

默认分别是Hive、MapReduce、HBase

1.5版本后可更改成自己需要的

## 主要特点

1. 标准SQL接口

    以Cube为核心，但是没有提东MDX接口，而是使用SQL作为对外服务接口

2. 支持超大数据集
   
   在数据集上规模上的局限主要在于维度的个数和基数

3. 亚秒级响应
   
   很多复杂的计算在离线的预计算中已完成

4. 可伸缩性和高吞吐率
   
   预计算降低了查询和计算的总量，是kylin在相同配置下可以承载更多并发查询

5. BI及可视化工具集成

    ODBC

    JDBC

    Rest API

    Zeppelin

# 二、快速入门

## 核心概念

数据仓库

OLAP：多维度分析、上卷、下钻、透视分析

维度和度量

事实表：存储事实记录的表

维度表：将事实表上重复出现的属性抽取、规范出来的一张表

使用维度表的好处：

1. 缩小事实表的大小

2. 便于维度表的管理和维护

3. 可以被多个事实表使用

Cube：数据立方体

Cuboid：某一种维度组合下计算的数据

Cube Segment：源数据中某一个片段计算出来的Cube

星形模型：维度表围绕着事实表，通过主键外键关联，维度表之间没有关联

维度表设计要求：

1. 主键必须唯一

2. 维度表会被加入内存，影刺越小越好

3. 改变频率低

4. 最好不要是Hive视图

Hive分区可以使kylin增量构建Cube，节省Cube构建时间，在kylin中叫分割时间列

维度的基数（Cardinality）：该维度在数据集中出现不同值的个数

超高基数的维度需要注意

# 三、增量构建

为了不重复计算历史数据而引入的功能

## Segment

将Cube划分为多个Segment，每个Segment用起始时间和结束时间来标志

Segment代表一段时间内源数据预计算结果

全量构建：Cube中只存在唯一一个Segment，该Segment没有分割时间的概念，不会区分历史数据和新加入的数据

增量构建的Cube上查询会比全量构建做更多运行时的聚合，因此，为保证查询性能，需要将某些Segment合并在一起

小数据Cube：全量构建

大数据Cube：增量构建

## 设计增量Cube

Cube的定义必须包含一个时间维度用来分割不同的Segment

分割的列必须是事实表上的列

若有两列

一列是日期 ==》 常规分割时间列

一列是时间 ==》 补充分割时间列

## Cube设置

Auto Merge Threshold：指定Segment自动合并的阈值

Retention Threshold：指定将过期的Segment自动抛弃

Partition Start Date：Cube默认的第一个Segment的起始时间

## 管理Cube碎片

增量构建中会导致Cube产生大量的Segment

Merge Segment

在Merge结束之前，不允许在这个Cube上进行其他构建任务，但是被选中的Segment仍然处于可用状态，Merge结束后被选中的Segment将被替换成新的Segment

Auto Merge

Auto Merge Threshold：指定Segment自动合并的阈值

允许设置几个层级的时间阈值

层级一般按升序排列

保留Segment

由于数据已经在Hive中已经有保留，无需在Kylin中再报留一份

Retention Threshold：指定将过期的Segment自动抛弃



# 四、流式构建

应对风控等对实时性要求较高的场景

kylin瞄准分钟级别的实时需求

## 准备流式数据

以消息流形式传递给流式构建引擎

消息必须包含：所有的维度信息、所有的度量信息、业务时间戳

不会直接把原始的业务时间戳作为一个维度，而会选择从它衍生出来的维度

消息队列默认选择kafka

由于schema不会经常变，所以会形成一张虚表

虚表中必须有一个timestamp类型





# 六、Cube优化

## Cuboid剪枝优化

kylin cube state reader 输出

最顶端Cuboid：base cuboid

每个cuboid都比父亲cuboid少1,

shrink：这个cuboid的行数与父亲节点的对比

膨胀率：当前cube的大小除以源数据大小的比率      0% ~ 1000% 之间

原因：

    1. 维度数量多

    2. 有基数过多的维度

## 剪枝优化工具

使用衍生维度：

不在事实表中加入衍生维度，这样会导致cuboid增长太快

使用聚合组：

同一个组内的维度更有可能被同一个sql用到，因此表现出更加紧密的内部关联

不同分组各自拥有一套维度集合

每个分组贡献出自己的cuboid，构建引擎会察觉出相同的cuboid，保证重复的cuboid只被物化一次

对于每个分组内部的维度，有三种方式定义他们的关系：

1. Mandatory：这个分组中每一个cuboid都会包含该维度，每个分组中可以有0、1、多个强制维度

2. Hierarchy：不同层级之间不应当有共享的维度

3. Joint：某些列形成一个联合，要么一起出现，要么一起不出现


## 并发粒度优化

将cuboid的数据分片到多个分区中，以实现cuboid数据读取的并行化，优化cube的查询速度

kylin.hbase.region.cut：决定segment需要几个分区存储，默认5.0，单位GB

kylin.hbase.region.count.min 和 kylin.hbase.region.count.max 调整最大最小分区


## Rowkeys优化

1. 编码

date

time

integer

dict

fixed_length

2. 按维度分片

hbase中不同region代表不同分片，这样hbase可以用协处理器在每个region进行预聚合


3. 调整Rowkeys顺序

在查询中被用作过滤条件的维度有可能放在其他维度的前面。

将经常出现在查询中的维度放在不经常出现的维度的前面。

对于基数较高的维度，如果查询会有这个维度上的过滤条件，那么将它往前调整；如果没有，则向后调整。


## 其他优化

1. 精度越高，占空间越大

2. 清理segment
























