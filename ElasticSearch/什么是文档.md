---
layout: post
#标题配置
title:  基本概念：索引、文档
#时间配置
date:   2019-12-30 20:00:00 +0800
#大类配置
categories: 数据搜索
#小类配置
tag: ElasticSearch
---

* content
{:toc}

## 说明
### 文档
+ 应用中的对象很少只是简单的键值列表，更多时候它拥有复杂的数据结构，比如包含日期、地理位置、另一个对象或者数组。
+ 总有一天你会想到把这些对象存储到数据库中。将这些数据保存到由行和列组成的关系数据库中，就好像是把一个丰富，信息表现力强的对象拆散了放入一个非常大的表格中：你不得不拆散对象以适应表模式（通常一列表示一个字段），然后又不得不在查询的时候重建它们。
+ Elasticsearch是面向文档(document oriented)的，这意味着它可以存储整个对象或文档(document)。然而它不仅仅是存储，还会索引(index)每个文档的内容使之可以被搜索。在Elasticsearch中，你可以对文档（而非成行成列的数据）进行索引、搜索、排序、过滤。这种理解数据的方式与以往完全不同，这也是Elasticsearch能够执行复杂的全文搜索的原因之一。

### 索引
+ 在Elasticsearch中存储数据的行为就叫做索引(indexing)

+ 在Elasticsearch中，文档归属于一种类型(type),而这些类型存在于索引(index)中，我们可以类比传统关系型数据库：
  + Relational DB -> Databases -> Tables -> Rows -> Columns
  + Elasticsearch -> Indices   -> Types  -> Documents -> Fields


+ Elasticsearch集群可以包含多个索引(indices)（数据库），每一个索引可以包含多个类型(types)（表），每一个类型包含多个文档(documents)（行），然后每个文档包含多个字段(Fields)（列）。

## Index 相关 API
```bash
#查看索引相关信息
GET kibana_sample_data_ecommerce

#查看索引的文档总数
GET kibana_sample_data_ecommerce/_count

#查看前10条文档，了解文档格式
POST kibana_sample_data_ecommerce/_search
{
}

#_cat indices API
#查看indices
GET /_cat/indices/kibana*?v&s=index

#查看状态为绿的索引
GET /_cat/indices?v&health=green

#按照文档个数排序
GET /_cat/indices?v&s=docs.count:desc

#查看具体的字段
GET /_cat/indices/kibana*?pri&v&h=health,index,pri,rep,docs.count,mt

#How much memory is used per index?
GET /_cat/indices?v&h=i,tm&s=tm:desc
```

创建员工目录，我们将进行如下操作：

    为每个员工的文档(document)建立索引，每个文档包含了相应员工的所有信息。
    每个文档的类型为employee。
    employee类型归属于索引megacorp。
    megacorp索引存储在Elasticsearch集群中。

通过一个命令执行完成的操作：
```bash
PUT /megacorp/employee/1
{
    "first_name" : "John",
    "last_name" :  "Smith",
    "age" :        25,
    "about" :      "I love to go rock climbing",
    "interests": [ "sports", "music" ]
}
```