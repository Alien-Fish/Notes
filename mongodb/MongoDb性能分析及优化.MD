### 1.查询方式优化，java版
禁止使用反射类结果集方式查询，反射消耗的性能很大，建议使用原生查询，查询出结果再转换成业务实体类。    
反例：  
List<DbEntity> list = mongoTemplate.aggregate(aggregate, collectionName, DbEntity.class).getMappedResults();  
正例:  
AggregateIterable<Document> aggregateIterable = mongoTemplate.getCollection(collectionName).aggregate(List<Bson>);  

### 2.count优化
禁止使用count计算总数，带条件查询建议使用CountDocuments，查文档集合总数使用EstimatedDocumentCount。  
实测：一千万数据下带条件CountDocuments查询耗时5s多，EstimatedDocumentCount估计0耗时。  

### 3.排序优化
排序字段不使用索引情况下，当超过了系统默认的32MB内存排序，会终止报错。   
开启allowDiskUse(true)或增大sort buffer。  
排序字段使用索引无需开启allowDiskUse，使用索引排序性能提高数百倍。  
实测：一千万数据下sort filed limit 1，无索引5s，有索引20ms。  

### 4.分页优化
分页采用skip+limit这种组合方式，小数据响应速度还可以，但是上千万的大数据，skip跳过大量的数据会很慢，并且随着数据库递增会越查越慢。  
通过条件查询+排序+限制返回记录，即边查询，边排序，排序之后，抽取上一页中的最后一条记录。  
实测：200万记录下查60万条记录，使用skip分页查询每次一万记录，从90ms一直递增5s左右。  
而使用最后一条记录比较方式则恒定在120ms左右，性能相差非常大。  

### 5.客户端连接数
查看客户端的连接信息，根据连接数降序排序  
netstat -nat | grep “27017” |awk ‘{print $5}’|awk -F: ‘{print $1}’|sort|uniq -c|sort -nr  

### 6.慢查询日志分析
--设置语句：

db.setProfilingLevel(1, { slowms: 4000 })  

--查询验证配置：

rs0:PRIMARY> db.getProfilingStatus()  

--查询级别：

rs0:PRIMARY> db.getProfilingLevel()  

1

--关闭设置Profiling：  

db.setProfilingLevel(0)  

--设置慢查询抽样比例：  

rs0:PRIMARY> db.setProfilingLevel(1, { sampleRate: 0.42 })  

--慢查询日志表，按查询时间降序  

db.system.profile.find.sort({ts:-1});  

--慢查询主要分析指标：

1).system.profile.op  

主要包含如下几类

insert

query

update

remove

getmore

command

代表了该慢日志的种类是什么，是查询、插入、更新、删除还是其他。

2).system.profile.ns

该项表明该慢日志是哪个库下的哪个集合所对应的慢日志。

3).system.profile.query

该项详细输出了慢日志的具体语句和行为

4).system.profile.keysExamined

该项表明为了找出最终结果MongoDB搜索了多少个key

5).system.profile.docsExamined

该项表明为了找出最终结果MongoDB搜索了多少个文档

6).system.profile.keyUpdates

该项表名有多少个index key在该操作中被更改，更改索引键也会有少量的性能消耗，因为数据库不单单要删除旧Key，还要插入新的Key到B-Tree索引中

7).system.profile.writeConflicts

写冲突发生的数量，例如update一个正在被别的update操作的文档

8).system.profile.numYield

为了让别的操作完成而屈服的次数，一般发生在需要访问的数据尚未被完全读取到内存中，MongoDB会优先完成在内存中的操作

9).system.profile.locks
Global
Represents global lock.

MMAPV1Journal
Represents MMAPv1 storage engine specific lock to synchronize journal writes; for non-MMAPv1 storage engines, the mode forMMAPV1Journal is empty.

Database
Represents database lock.

Collection
Represents collection lock.

Metadata
Represents metadata lock.

oplog
Represents lock on the oplog.

10).system.profile.locks.acquireCoun

在各种不用的种类下，请求锁的次数

11).system.profile.nreturned

该操作最终返回文档的数量

12).system.profile.responseLength

结果返回的大小，单位为bytes，该值如果过大，则需考虑limit()等方式减少输出结果

13).system.profile.millis

该操作从开始到结束耗时多少，单位为毫秒

14).system.profile.execStats

包含了一些该操作的统计信息，只有query类型的才会显示

15).system.profile.execStats.stage

包含了该操作的详细信息，例如是否用到索引

16).system.profile.ts

该操作执行时的时间

17).system.profile.client

哪个客户端发起的该操作，并显示出该客户端的ip或hostname

18).system.profile.allUsers

哪个认证用户执行的该操作

19).system.profile.user

是否认证用户执行该操作，如认证后使用其他用户操作，该项为空

实测：200万数据下，批次查询60几万数据耗时分析


### 7.执行计划分析
stage常见的操作描述如下：
- COLLSCAN 集合扫描
- IXSCAN 索引扫描
- FETCH 检出文档
- SHARD_MERGE 合并分片中结果
- SHARDING_FILTER 分片中过滤掉孤立文档
- LIMIT 使用limit 限制返回数
- PROJECTION 使用 skip 进行跳过
- IDHACK 针对_id进行查询
- COUNT 利用db.coll.explain().count()之类进行count运算
- COUNTSCAN count不使用Index进行count时的stage返回
- COUNT_SCAN count使用了Index进行count时的stage返回
- SUBPLA 未使用到索引的$or查询的stage返回
- TEXT 使用全文索引进行查询时候的stage返回
- PROJECTION 限定返回字段时候stage的返回

explain 希望看到的阶段
- Fetch+IDHACK
- Fetch+ixscan
- Limit+（Fetch+ixscan）
- PROJECTION+ixscan
- SHARDING_FILTER+ixscan
- COUNT_SCAN

explain 不希望看到的阶段
- COLLSCAN（全表扫描）
- SORT（使用sort但是无index）
- 不合理的SKIP，
- SUBPLA（未用到index的$or）
- COUNTSCAN