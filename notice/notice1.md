# Druid架构及原理

https://my.oschina.net/hblt147/blog/3004623?device=geekTime.ios

## Segment: 

Druid中有个重要的数据单位叫segment，其是Druid通过bitmap indexing从raw data生成的（batch or realtime）。segment保证了查询的速度。可以自己设置每个segment对应的数据粒度，这个应用中广告流量查询的最小粒度是天，所以每天的数据会被创建成一个segment。注意segment是不可修改的，如果需要修改，只能够修改raw data，重新创建segment了。

#### Druid本身包含5个组成部分：Broker nodes, Historical nodes, Realtime nodes, Coordinator Nodes和indexing services. 

分别的作用如下：

- Broker nodes: 负责响应外部的查询请求，通过查询Zookeeper将请求划分成segments分别转发给Historical和Real-time nodes，最终合并并返回查询结果给外部；
- Historial nodes: 负责’Historical’ segments的存储和查询。其会从deep storage中load segments，并响应Broder nodes的请求。Historical nodes通常会在本机同步deep storage上的部分segments，所以即使deep storage不可访问了，Historical nodes还是能serve其同步的segments的查询；
- Real-time nodes: 用于存储和查询热数据，会定期地将数据build成segments移到Historical nodes。一般会使用外部依赖kafka来提高realtime data ingestion的可用性。如果不需要实时ingest数据到cluter中，可以舍弃Real-time nodes，只定时地batch ingestion数据到deep storage；
- Coordinator nodes: 可以认为是Druid中的master，其通过Zookeeper管理Historical和Real-time nodes，且通过Mysql中的metadata管理Segments
- Druid中通常还会起一些indexing services用于数据导入，batch data和streaming data都可以通过给indexing services发请求来导入数据。

#### Druid还包含3个外部依赖

- Mysql：存储Druid中的各种metadata（里面的数据都是Druid自身创建和插入的），包含3张表：”druid_config”（通常是空的）, “druid_rules”（coordinator nodes使用的一些规则信息，比如哪个segment从哪个node去load）和“druid_segments”（存储每个segment的metadata信息）；
- Deep storage: 存储segments，Druid目前已经支持本地磁盘，NFS挂载磁盘，HDFS，S3等。Deep Storage的数据有2个来源，一个是batch Ingestion, 另一个是real-time nodes；
- ZooKeeper: 被Druid用于管理当前cluster的状态，比如记录哪些segments从Real-time nodes移到了Historical nodes；

Druid的查询是通过给Broker Nodes发送HTTP POST请求（也可以直接给Historical or Realtime Node），具体可见Druid官方文档。查询条件的描述是json文件，查询的response也是json格式。Druid的查询包含如下4种：

- Time Boundary Queries: 用于查询全部数据的时间跨度
- groupBy Queries: 是Druid的最典型查询方式，非常类似于Mysql的groupBy查询。query body中几个元素可以这么理解：
  - “aggregation”: 对应mysql”select XX from”部分，即你想查哪些列的聚合结果; 
  - “dimensions”: 对应mysql”group by XX”，即你想基于哪些列做聚合;
  - “filter”: 对应mysql”where XX”条件，即过滤条件；
  - “granularity”: 数据聚合的粒度;
- Timeseries queries: 其统计满足filter条件的”rows”上某几列的聚合结果，相比”groupBy Queries”不指定基于哪几列进行聚合，效率更高;
- TopN queries: 用于查询某一列上按照某种metric排序的最常见的N个values;

## 总结

1.Druid是一个开源的，分布式的，列存储的，适用于实时数据分析的系统，文档详细，易于上手；
  - Druid在设计时充分考虑到了Highly Available，各种nodes挂掉都不会使得druid停止工作（但是状态会无法更新）；
  - Druid中的各个components之间耦合性低，如果不需要streaming data ingestion完全可以忽略realtime node；
  - Druid的数据单位Segment是不可修改的，我们的做法是生成新的segments替换现有的；
  - Druid使用Bitmap indexing加速column-store的查询速度，使用了一个叫做CONCISE的算法来对bitmap indexing进行压缩，使得生成的segments比原始文本文件小很多；
  
2.在我们的应用场景下（一共10几台机器，数据大概100列，行数是亿级别），平均查询时间<2秒，是同样机器数目的Mysql cluter的1/100 ~ 1/10；

3.Druid的一些“局限”：
  - Segment的不可修改性简化了Druid的实现，但是如果你有修改数据的需求，必须重新创建segment，而bitmap indexing的过程是比较耗时的；
  - Druid能接受的数据的格式相对简单，比如不能处理嵌套结构的数据

# DATA
````
  timestamp             publisher          advertiser  gender  country  click  price
  2011-01-01T01:01:35Z  bieberfever.com    google.com  Male    USA      0      0.65
  2011-01-01T01:03:63Z  bieberfever.com    google.com  Male    USA      0      0.62
  2011-01-01T01:04:51Z  bieberfever.com    google.com  Male    USA      1      0.45
  2011-01-01T01:00:00Z  ultratrimfast.com  google.com  Female  UK       0      0.87
  2011-01-01T02:00:00Z  ultratrimfast.com  google.com  Female  UK       0      0.99
  2011-01-01T02:00:00Z  ultratrimfast.com  google.com  Female  UK       1      1.53
````

熟悉OLAP的同学，对以下这些概念一定不陌生，druid也把数据分为以下三个部分：

Timestamp Column：将时间单独处理，是因为druid所有的操作都是围绕时间轴来进行的。

Dimension Columns：维度字段，是数据的属性， 一般被用来过滤数据。上面的例子，我们有四个维度, publisher, advertiser, gender, country.  他们每一个都可以看是数据立方体的一个轴，都可以用来用来做横切。

Metric Columns: 度量字段，是用来做聚合或者相关计算的。 上边的数据， click和price是俩个度量。度量是可以衡量的数据，一般可以有如下的操作，count ，sum等等

## ROLL-UP
roll-up （上卷）是olap的基本操作(除此之外还有下钻，切片等， 基本理论是一样的)。  在数据统计里，由于数据量太多，一般对细分的数据不是特别干兴趣，或者说没有太大关注的意义。但是按照维度的汇总或者统计，确实很有用的。druid通过一个roll-up的处理，将原始数据在注入的时候就进行汇总处理。roll-up 是在维度过滤之前的第一层聚合操作，如下：
```
GROUP BY timestamp, publisher, advertiser, gender, country
  :: impressions = COUNT(1),  clicks = SUM(click),  revenue = SUM(price)
```
聚合后数据就变成了如下的样子
```
timestamp             publisher          advertiser  gender country impressions clicks revenue
 2011-01-01T01:00:00Z  ultratrimfast.com  google.com  Male   USA     1800        25     15.70
 2011-01-01T01:00:00Z  bieberfever.com    google.com  Male   USA     2912        42     29.18
 2011-01-01T02:00:00Z  ultratrimfast.com  google.com  Male   UK      1953        17     17.31
 2011-01-01T02:00:00Z  bieberfever.com    google.com  Male   UK      3194        170    34.01
```
