我Google了下，大致给出的答案如下：

1. 使用bulk API
2. 初次索引的时候，把 replica 设置为 0
3. 增大 threadpool.index.queue_size
4. 增大 indices.memory.index_buffer_size
5. 增大 index.translog.flush_threshold_ops
6. 增大 index.translog.sync_interval
7. 增大 index.engine.robin.refresh_interval 

这篇文章会讲述上面几个参数的原理，以及一些其他的思路。这些参数大体上是朝着两个方向优化的：

1. 减少磁盘写入
2. 增大构建索引处理资源

一般而言，通过第二种方式的需要慎用，会对集群查询功能造成比较大的影响。
这里还有两种形态的解决方案：

1. 关闭一些特定场景并不需要的功能，比如Translog或者Version等
2. 将部分计算挪到其他并行计算框架上，比如数据的分片计算等，都可以放到Spark上事先算好

## 上面的参数都和什么有关

其中  5,6 属于 TransLog 相关。
4 则和Lucene相关
3 则因为ES里大量采用线程池，构建索引的时候，是有单独的线程池做处理的
7 的话个人认为影响不大
2 的话，能够使用上的场景有限。个人认为Replica这块可以使用Kafka的ISR机制。所有数据还是都从Primary写和读。Replica尽量只作为备份数据。



## Translog

为什么要有Translog? 因为Translog顺序写日志比构建索引更高效。我们不可能每加一条记录就Commit一次，这样会有大量的文件和磁盘IO产生。但是我们又想避免程序挂掉或者硬件故障而出现数据丢失，所以有了Translog，通常这种日志我们叫做Write Ahead Log。

为了保证数据的完整性，ES默认是每次request结束后都会进行一次sync操作。具体可以查看如下方法：
      
    org.elasticsearch.action.bulk.TransportShardBulkAction.processAfter 

该方法会调用IndexShard.sync 方法进行文件落地。

你也可以通过设置`index.translog.durability=async` 来完成异步落地。这里的异步其实可能会有一点点误导。前面是每次request结束后都会进行sync,这里的sync仅仅是将Translog落地。而无论你是否设置了async,都会执行如下操作：

根据条件，主要是每隔sync_interval(5s) ，如果flush_threshold_ops(Integer.MAX_VALUE)，flush_threshold_size(512m),flush_threshold_period(30m)  满足对应的条件，则进行flush操作，这里除了对Translog进行Commit以外，也对索引进行了Commit。 

所以如果你是海量的日志，可以容忍发生故障时丢失一定的数据，那么完全可以设置，`index.translog.durability=async`，并且将前面提到的flush*相关的参数调大。

而极端情况，你还可以有两个选择：

1. 设置`index.translog.durability=async`，接着设置`index.translog.disable_flush=true`进行禁用定时flush。然后你可以通过应用程序自己手动来控制flush。

2. 通过改写ES 去掉Translog日志相关的功能


## Version

 Version可以让ES实现并发修改，但是带来的性能影响也是极大的,这里主要有两块：

1. 需要访问索引里的版本号，触发磁盘读写
2. 锁机制

目前而言，似乎没有办法直接关闭Version机制。你可以使用自增长ID并且在构建索引时，index 类型设置为create。这样可以跳过版本检查。

这个场景主要应用于不可变日志导入，随着ES被越来越多的用来做日志分析，日志没有主键ID,所以使用自增ID是合适的，并且不会进行更新，使用一个固定的版本号也是合适的。而不可变日志往往是追求吞吐量。

当然，如果有必要，我们也可以通过改写ES相关代码，禁用版本管理。

##  分发代理

ES是对索引进行了分片(Shard)，然后数据被分发到不同的Shard。这样 查询和构建索引其实都存在一个问题：

 > 如果是构建索引，则需要对数据分拣，然后根据Shard分布分发到不同的Node节点上。
 > 如果是查询，则对外提供的Node需要收集各个Shard的数据做Merge

这都会对对外提供的节点造成较大的压力，从而影响整个bulk/query 的速度。

一个可行的方案是，直接面向客户提供构建索引和查询API的Node节点都采用client模式，不存储数据，可以达到一定的优化效果。

另外一个较为麻烦但似乎会更优的解决方案是，如果你使用类似Spark Streaming这种流式处理程序，在最后往ES输出的时候，可以做如下几件事情：

1. 获取所有primary shard的信息，并且给所有shard带上一个顺序的数字序号，得到partition(顺序序号) -> shardId的映射关系
2. 对数据进行repartition,分区后每个partition对应一个shard的数据
3. 遍历这些partions,写入ES。方法为直接通过RPC 方式，类似`transportService.sendRequest` 将数据批量发送到对应包含有对应ShardId的Node节点上。

这样有三点好处：

1. 所有的数据都被直接分到各个Node上直接处理。避免所有的数据先集中到一台服务器
2. 避免二次分发，减少一次网络IO
3. 防止最先处理数据的Node压力太大而导致木桶短板效应

## 场景

因为我正好要做日志分析类的应用，追求高吞吐量，这样上面的三个优化其实都可以做了。一个典型只增不更新的日志入库操作，可以采用如下方案：

1. 对接Spark Streaming,在Spark里对数据做好分片，直接推送到ES的各个节点
2. 禁止自动flush操作，每个batch 结束后手动flush。
3. 避免使用Version

我们可以预期ES会产生多少个新的Segment文件，通过控制batch的周期和大小，预判出ES Segment索引文件的生成大小和Merge情况。最大可能减少ES的一些额外消耗

## 总结

大体是下面这三个点让es比原生的lucene吞吐量下降了不少：

1.  为了数据完整性  ES额外添加了WAL(tanslog)
2.  为了能够并发修改  添加了版本机制
3.  对外提供服务的node节点存在瓶颈

ES的线性扩展问题主要受限于第三点,
具体描述就是：

>如果是构建索引，接受到请求的Node节点需要对数据分拣，然后根据Shard分布分发到不同的Node节点上。
如果是查询，则对外提供的Node需要收集各个Shard的数据做Merge

另外，索引的读写并不需要向Master汇报。
