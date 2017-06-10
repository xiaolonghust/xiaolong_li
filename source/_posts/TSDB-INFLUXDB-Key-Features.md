---
title: InfluxDB的关键特性
date: 2017-06-09 19:26:23
tags: 
  - 时序数据
  - InfluxDB
categories: 数据库
---

## Retention Policies

Retention Policies - 数据保留策略是用来定义数据的存放时长，或者是保留某一时间段的数据。每个数据库可以有多个数据保留策略，但只能有一个默认策略。

InfluxDB本身是不支持对数据的删除操作，时序数据通常对历史数据的保留时间间隔是有规定的，例如一个线上时序数据业务，可能只需要保留最近1周的数据。为了方便使用，时序数据库必须有数据自动rotate的能力。定义数据保留策略是要让InfluxDB知道在什么时候可以丢弃那些数据，从而达到简单高效的处理数据。

<!-- more -->

### 创建策略
在InfluxDB中，定义一个PILICY的语句：

```
CREATE RETENTION POLICY "two_hours" ON "food_data" DURATION 2h REPLICATION 1 DEFAULT
```
一个策略包括：

NAME--名称，此示例名称为==two_hours==

DURATION--持续时间，0代表无限制，此示例为2小时

shardGroupDuration--shardGroup的存储时间，shardGroup是InfluxDB的一个基本储存结构，应该大于这个时间的数据在查询效率上应该有所降低。

replicaN--全称是REPLICATION，副本个数

DEFAULT--是否是默认策略

上述语句的含义是：

> 在food_data库上建立一个名为two_hours的策略，该策略的数据保留时间为2h，副本集数为1，该条策略为food_data库的默认策略

### 修改策略

```
ALTER RETENTION POLICY "two_hours" ON "food_data" DURATION 4h DEFAULT
```
> 示例的作用是将策略"two_hours"的数据保留时间修改为4h

### 删除策略


```
DROP retention POLICY "two_hours" ON "food_data"
```
> 示例是可以将food_data库上的two_hours策略删除


## Continuous Queries

Continuous Queries - 连续查询算是InfluxDB的一个大杀器。连续查询是在数据不断进来以后，按指定的查询语句定时地、不断地做聚合，然后放到一个新的系列里面。实际上后面就可以不需要直接在原始数据里面查询，而直接查询这个Continuous Queries生成新的序列。这样查询效率就会快很多，数据只需要在后台不停的算，不停地聚合，从而达到提高查询效率的效果。
（注：根据InfluxDB的存储结构，InfluxDB牺牲了读效率而提高了写效率）

使用连续查询是最优的降低采样率的方式，连续查询和存储策略搭配使用将会大大降低InfluxDB的系统占用量。而且使用连续查询后，数据会存放到指定的数据表中，这样就为以后统计不同精度的数据提供了方便。

### 创建一个连续查询


```
CREATE CONTINUOUS QUERY "cq_30m" ON "logInfo" BEGIN SELECT mean(utm) INTO "mem_utm_30m" FROM mem GROUP BY time(30m) END
```
> 示例在food_data库中新建了一个名为cq_30m 的连续查询，每30m计算一个utm字段的平均值，插入mem_utm_30m表中


### 显示所有连续查询

```
SHOW CONTINUOUS QUERIES
```

### 删除连续查询

```
DROP CONTINUOUS QUERY "cq_30m" ON "logInfo"
```
> 示例是将logInfo库上名为cq_30的连续删除


## 自带管理工具

安装启动完InfluxDB后，可以通过本地8083端口访问web管理页面

> 官方在1.1.0版本以后，默认关闭了web管理页面

如遇到web打不开的情况，使用如下解决方案

### 解决方案

打开InfluxDB的配置文件，找到下面几行

```
### NOTE: This interface is deprecated as of 1.1.0 and will be removed in a future release.
[admin]
  # Determines whether the admin service is enabled.
  # enabled = false

  # The default bind address used by the admin service.
  # bind-address = ":8083"

  # Whether the admin service should use HTTPS.
  # https-enabled = false

  # The SSL certificate used when HTTPS is enabled.
  # https-certificate = "/etc/ssl/influxdb.pem"

###
```
修改enabled = true，bind-address = ":8083"

并且由配置文件中的
> NOTE: This interface is deprecated as of 1.1.0 and will be removed in a future release.

可以看出在1.1.0版本以后默认关闭了web管理页面，在不久的将来这个功能还将被移除。

本人觉得这个管理页面很人性化，对移除这个功能表示不理解，或许后面会加入新的功能来替代它。就请继续关注吧！