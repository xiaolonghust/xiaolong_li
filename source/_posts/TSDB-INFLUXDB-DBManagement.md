---
title: InfluxDB数据库管理
date: 2017-06-17 11:43:43
tags: 
  - 时序数据
  - InfluxDB
categories: 数据库
---

> 基于InfluxDB 1.2版本

## 数据管理
### 创建数据库

```
CREATE DATABASE <database_name> [WITH [DURATION <duration>] [REPLICATION <n>] [SHARD DURATION <duration>] [NAME <retention-policy-name>]]
```
#### 语法解析
和其他数据库一样，使用CREATE DATABASE去创建一个数据库。
database_name是数据库名称。
创建数据库时可以为该数据库创建数据保留策略，WITH, DURATION, REPLICATION, SHARD DURATION, 和 NAME是用来创建新的策略，如果没有指定创建数据保留策略，则会使用默认策略，InfluxDB默认会为每一个数据库创建一个名为autogen的数据保留策略。

<!-- more -->

#### Example 1：创建一个数据库
```
CREATE DATABASE "NOAA_water_database"
```
这条语句会创建一个名为NOAA_water_database的数据库，并使用默认的数据保留策略

#### Example 2：创建一个指定数据保留策略的数据库
```
CREATE DATABASE "NOAA_water_database" WITH DURATION 3d REPLICATION 1 SHARD DURATION 1h NAME "liquid"
```
这条语句会创建一个名为NOAA_water_database的数据库，并为其指定一个[过期时间为3天，副本集数为1，SHARDGROUP时间为1小时，名为liquid]的数据保留策略。

> 当成功执行一个创建数据库语句时，InfluxDB会返回一个空的结果。并且当试图去创建一个已经存在的数据库时，InfluxDB什么都不会做，当然也不会返回错误信息。

### 删除数据库

```
DROP DATABASE <database_name>
```
#### 语法说明
当执行删除数据库的操作成功时，会删除指定数据库中的所有信息，包括 measurements, series, continuous queries, 和 retention policies等。

#### Example：删除数据库
```
DROP DATABASE "NOAA_water_database"
```
这条语句会删除名为NOAA_water_database的数据库。

> 当成功执行删除数据库的操作时，InfluxDB会返回一个空的结果。并且当试图去删除一个不存在的数据库时，InfluxDB什么都不会做，当然也不会返回错误信息。

### 通过DROP SERIES删除数据
> 通过DROP SERIES可以删除一个数据库来自同一个Series的所有Point，并从索引中删除Series。但是它不支持在where从句中使用时间区间[time intervals]。

```
DROP SERIES FROM <measurement_name[,measurement_name]> WHERE <tag_key>='<tag_value>'
```
#### 语法说明
当这条语句执行成功时，会删除所有符合条件的Point

#### Example 1：删除一个表中所有Series
```
DROP SERIES FROM "h2o_feet"
```
这条语句执行成功时，会删除h2o_feet中所有序列

#### Example 2：通过指定tag删除一个表中的Series
```
DROP SERIES FROM "h2o_feet" WHERE "location" = 'santa_monica'
```
这条语句执行成功时，会删除h2o_feet中location为santa_monica的所有序列

#### Example 3：删除数据库中所有Measurement表中指定tag的所Series
```
DROP SERIES WHERE "location" = 'santa_monica'
```
这条语句执行成功时，会删除库中包含所有表中location为santa_monica的所有序列

> 成功执行DROP SERIES语句返回一个空的结果

### 通过DELETE SERIES删除数据
> 通过DELETE可以删除一个数据库来自同一个Series的所有Point，与DROP SERIES不同，它不能从索引中删除Series。但是它支持在where从句中使用时间区间[time intervals]。

```
DELETE FROM <measurement_name> WHERE [<tag_key>='<tag_value>'] | [<time interval>]
```
#### 语法说明
当这条语句执行成功时，会删除所有符合条件的Point，它支持在WHERE从句中使用时间区间[time intervals]

#### Example 1：删除表中所有数据
```
DELETE FROM "h2o_feet"
```
这条语句执行成功后，会清空表h2o_feet中所有数据，并且删除表h2o_feet

#### Example 2：通过指定tag值删除数据
```
DELETE FROM "h2o_quality" WHERE "randtag" = '3'
```
这条语句执行成功后，会删除表h2o_quality中randtag为3的所有数据

#### Example 3：通过指定时间区间删除数据
```
DELETE WHERE time < '2016-01-01'
```
这条语句执行成功后，会删除数据库中在2016-01-01之前的所有数据

> DELETE语句执行成功后会返回一个空的结果

#### 关于DELETE要注意的几点：
1. 当指定Measurement时，在WHERE从句中使用tag进行操作时，可以使用正则表达式
2. DELETE语句在WHERE从句中不支持使用Field字段
3. 

### 删除表
> 通过DROP MEASUREMENT可以从指定的Measurement表中删除所有数据以及Series，并且从索引中删除measuerment

```
DROP MEASUREMENT <measurement_name>
```
#### 语法说明
通过制定Measurement名称删除表中所有数据

#### Example：删除指定表
```
DROP MEASUREMENT "h2o_feet"
```
这条语句执行成功后，可以删除h2o_feet中所有数据，并且从索引中删除

> DROP MEASUREMENT语句执行成功后会返回一个空的结果

#### 关于DROP MEASUREMENT要注意的几点：
1. DROP MEASUREMENT可以删除指定Measurement中的所有数据以及Series，但是不能删除与之相关的连续查询[continuous queries]
2. 目前， DROP MEASUREMENT删除指定Measurement时，不能使用正则表达式

### DROP SHARD
> 通过DROP SHARD可以删除一个shard，也可以从元数据中删除这个shard

```
DROP SHARD <shard_id_number>
```
#### 语法说明
通过制定shard_id来删除指定shard

#### Example：
```
DROP SHARD 1
```
这条语句执行成功后，可以删除id为1的shard，并从元数据中删除该shard

> DROP SHARD语句执行成功后会返回一个空的结果，即使指定的shard_id不存在，InfluxDB也不会返回一个错误信息

## 数据保留策略[RETENTION POLICY]

RETENTION POLICY : 数据保留策略，也叫数据过期策略，是InfluxDB的一个非常重要的概念。数据保留策略是用来定义数据的存放时长，或者是保留某一时间段的数据。每个数据库可以有多个数据保留策略，但只能有一个默认策略。定义数据保留策略是要让InfluxDB知道在什么时候可以丢弃那些很久以前数据，从而达到简单高效的处理数据。

当新创建一个数据库时，InfluxDB会自动为该数据库创建一个名为aotugen的不限过期时间的策略。你也可以在InfluxDB的配置文件中制定这个默认策略的名称，或者禁用自动创建。

### 创建一个策略
```
CREATE RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> [SHARD DURATION <duration>] [DEFAULT]
```
#### 语法说明
使用CREATE RETENTION POLICY可以创建一个指定策略名[retention_policy_name]，在一个指定的数据库[database_name]上，指定数据保留时间[duration]，副本集数[n]，SHARDGROUP时间[SHARD DURATION <duration>]，以及是否要作为该数据库的默认策略[DEFAULT]，其中，在创建策略时，SHARD DURATION时间和是否是默认策略可以不指定。

##### DURATION
DURATION定义了在InfluxDB上数据可以保留的时间，是一个时间区域或者是无限期。最小的保留时间是1小时[1h]，最大的保留时间为无限期[0]。该字段单位可以是小时(如2h)，也可以是天(如3d)，当该字段值为0时，表示数据无过期时间，将无限期保留。

##### REPLICATION
REPLICATION定义了在InfluxDB上数据的副本集数，n表示副本集节点的个数。
> 目前使用InfluxDB都是单实例

##### SHARD DURATION
SHARD DURATION定义在InfluxDB上一个shardGroup上数据的时间范围，但是不支持无限制时间范围，这是一个可选项，如果不指定，它会根据数据保留时间确定其时间范围，具体对应关键见下表：

Retention Policy’s DURATION | Shard Group Duration
---|---
< 2day | 1h
>= 2 days and <= 6 months | 1 day
> 6 months | 7 days

SHARD DURATION最小被允许的值是1h。
如果在创建一个策略时，试图将SHARD DURATION的值设为(0s, 1h)区间的值，InfluxDB会自动将其设置为1h。
如果在创建一个策略时，试图将SHARD DURATION的值设为0s，InfluxDB会自动按上述默认值设置。

#### Example 1：创建策略

```
CREATE RETENTION POLICY "one_day_only" ON "NOAA_water_database" DURATION 1d REPLICATION 1
```
执行这条语句，将会为名为NOAA_water_database的数据库，创建一个名为one_day_only的数据保留策略，该策略数据保留时间为一天，副本集数为1

#### Example 2：创建默认策略

```
CREATE RETENTION POLICY "one_day_only" ON "NOAA_water_database" DURATION 23h60m REPLICATION 1 DEFAULT
```
执行这条语句，会创建一个和Example 1一样的策略，但是该策略将作为数据库NOAA_water_database的默认策略

> 成功创建一个策略会返回一个空的结果。如果试图去创建一个已经存在并且一样的策略，InfluxDB不会返回错误的结果。但是如果试图去创建一个策略名存在，属性不一样的策略时，这时InfluxDB会返回一个错误的结果[Server returned error: retention policy already exists]。

### 修改策略
```
ALTER RETENTION POLICY <retention_policy_name> ON <database_name> DURATION <duration> REPLICATION <n> SHARD DURATION <duration> DEFAULT
```
#### 语法说明
使用ALTER RETENTION POLICY修改一个策略时，必须至少指定修改策略属性[DURATION, REPLICATION, SHARD DURATION, DEFAULT]中的一个。

#### Example：修改策略
首先创建一个策略:
```
CREATE RETENTION POLICY "what_is_time" ON "NOAA_water_database" DURATION 2d REPLICATION 1
```
这条语句为数据库NOAA_water_database建立一个名为what_is_time，数据保留时间为2天，副本集数为1的策略。
```
ALTER RETENTION POLICY "what_is_time" ON "NOAA_water_database" DURATION 7d SHARD DURATION 2h DEFAULT
```
这条语句将数据库NOAA_water_database上的策略what_is_time的数据保留时间修改为7天，SHARD DURATION修改为2小时，并且作为将数据库NOAA_water_database默认策略。

> ALTER RETENTION POLICY语句执行成功后，InfluxDB会返回一个空的结果。如果试图去修改一个不存在的策略，InfluxDB会返回错误信息[Server returned error: retention policy not found: what_is_time_2]。

### 删除策略
```
DROP RETENTION POLICY <retention_policy_name> ON <database_name>
```
#### 语法说明
DROP RETENTION POLICY可以删除指定数据库上的指定策略

#### Example：删除指定策略
```
DROP RETENTION POLICY "what_is_time" ON "NOAA_water_database"
```
执行这条语句，会将数据库NOAA_water_database上名为what_is_time的策略删除

> DROP RETENTION POLICY语句执行成功后，InfluxDB会返回一个空的结果。如果试图去删除一个不存在的策略，InfluxDB也不会返回错误的信息。
