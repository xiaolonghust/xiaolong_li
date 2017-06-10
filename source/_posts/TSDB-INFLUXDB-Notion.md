---
title: InfluxDB的基本概念
date: 2017-06-09 13:06:47
tags: 
  - 时序数据
  - InfluxDB
categories: 数据库
---

## InfluxDB的基本概念

InfluxDB是Go语言开发的一个开源分布式时序数据库，非常适合存储指标、事件、分析等数据。

### DATABASE

Database - 是数据库最基本的概念，InfluXDB作为时序数据库也不例外。

### MEASUREMENT

Measurement - 数据表，在InfluxDB中的作用类似传统数据库的table。

<!-- more -->

### TAG/TAGS

Tag - 标签，在InfluxDB中，Tag是非常重要的概念，在Measurement中存在，并和Measurement一起组成数据库的索引，每个Measurement中可以有多个Tag组成Tags。 索引是以键值对“key-value”形式存在。

### FIELD

Field - 数据，field主要是用来存放数据部分，也是以键值对“key-value”形式存在。

### TIMESTAMP

Timestamp - 时间戳，是时序数据库最重要的概念，没有之一。一条数据可以没有Tag，也可以没有Field，但是绝对不能没有Timestamp，可以插入数据时指定，也可以由系统生成。

### POINT

Point - 点，InfluxDB特有的概念，表示Measurement表中的一行数据，表示每个表里某个时刻的某个条件下的数据，因为体现在图表上就是一个点，于是将其称为Point。一个Point由Timestamp(时间戳)、Tag(标签)、Filed(数据)组成。

### SERIES

Series - 序列，所有在数据库中的数据，都需要通过图表来表示，Series表示这个表里面的所有的数据可以在图标上画成几条线。
（注：线条的个数由tags排列组合计算出来）


## 示例

### 数据表

![image](/images/tsdb/measurement.jpg)


### 序列

![image](/images/tsdb/series.jpg)