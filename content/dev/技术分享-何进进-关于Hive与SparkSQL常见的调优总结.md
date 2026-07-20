---
title: 关于 Hive 与 SparkSQL 常见的调优总结
date: 2026-07-20
description: 梳理 Hive 与 SparkSQL 两大场景下最常见的性能调优手段——小文件问题与数据倾斜，涵盖参数配置、SQL 写法与执行引擎层面的解决思路
toc: true
slug: hive-spark-sql-tuning-summary
categories:
    - Dev
    - 技术分享
    - 大数据
tags:
    - hive
    - sparksql
    - 调优
    - 小文件
    - 数据倾斜
---

# 🐘 关于 Hive 与 SparkSQL 常见的调优总结

---

## 一、小文件问题

在基于 Hadoop 的大数据场景下，当存在大量的小文件时会出现性能上的问题。一般来说，小文件的大小远远小于 HDFS 的存储块大小（128MB），当我们用 Hive 或者 Spark 处理这些小文件时，会出现下面的问题：

1. **存储开销**：小文件会增加存储的开销，因为每个文件都会占用磁盘上的存储空间和文件系统的元数据
2. **NameNode 负载**：在 HDFS 中，NameNode 用于维护文件系统的元数据，如果此时有大量的小文件，会增加 NameNode 元数据的负载，导致性能下降更有可能崩溃
3. **读写性能**：对于计算引擎来说，处理小文件会增加文件系统 IO 操作，导致读写性能下降
4. **任务启动开销**：大量的小文件，可能会增加任务的启动和调度开销
5. **内存开销**：Spark 中的 Executor 会为每个任务分配一定的内存，如果有大量小文件，会增加 Executor 的内存开销
6. **并行度受限**：小文件可能会限制并行处理的能力，因为每个文件都需要一个任务来处理，如果任务数量过多，会降低并行度

### 1.1 为什么会产生小文件？

1. 动态分区插入数据
2. 数据源本身就有很多小文件
3. 对于 MapReduce 来说，reduce 的数量越多，小文件也越多，输出的文件个数与 reduce 个数一致
4. 像是 SparkSQL，执行时会将文件切分，如果设置的并行度不合理，会导致每个分区的数据量很小，就产生大量小文件
5. 数据写入之前，没有做聚合、合并的操作，这样文件就不会合并，就会产生大量小文件，例如我只有很多 Map 过程，而没有 Reduce 过程

### 1.2 解决方法

**对于 Spark 来说**，首先是利用参数：

- `spark.sql.files.openCostInBytes`：用相同时间内可以扫描的数据的大小来衡量打开一个文件的开销，将多个文件写入同一分区时有用，默认 4M，理论上高估会好。其实就是将小于这个数的文件合并到一个分区中的意思，并且该参数仅对 Parquet、ORC、JSON 文件有效
- `spark.sql.files.maxPartitionBytes`：打包传入一个分区的最大字节，在读取文件的时候，默认 128M，也是仅对 Parquet、ORC、JSON 文件有效

其次，在编写 SQL 代码时，添加 hint、`repartition` 或者 `coalesce`。注意，`repartition` 会产生 shuffle，但是既可以降低分区数也可以增加分区数；而 `coalesce` 只能将分区数变多，所以，如果是合并小文件，我们可以使用 `repartition`，样例如下：

```sql
insert into table a
select /*+ repartition(10) */ id, name, address from b
```

一般来说，上面两个参数非必要不用改，建议直接在最终插入表的时候进行重分区。

**对于 Hive 来说**，导致的原因：文件数量 = Reduce 的 task 数量 × 分区数；文件数量 = Map 的 task 数量 × 分区数。

- 如果是 ORC 或 RC 表，并且是内表，可以执行：

```sql
ALTER TABLE table_name
  [PARTITION (partition_key = 'partition_value' [, ...])]
  CONCATENATE;
```

- map 前合并、map 输出段合并、reduce 输出段合并，对应的参数：

```properties
hive.input.format=org.apache.hadoop.hive.ql.io.CombineHiveInputFormat   执行 map 前进行小文件合并
mapred.max.split.size=256000000                                         每个 map 最大输入大小
mapred.min.split.size.per.node=10000000                                 一个节点上 split 的至少的大小
hive.merge.mapfiles=true                                                设置 map 端输出进行合并，默认是 true
hive.merge.mapredfiles=true                                             设置 reduce 端输出进行合并，默认是 false
hive.merge.size.per.task=256000000                                      设置合并文件的大小
hive.merge.smallfiles.avgsize=16000000                                  当输出文件的平均大小小于该值时，启动一个独立的 mapreduce 任务进行文件 merge
hive.exec.compress.output=true                                          hive 的查询结果输出是否进行压缩
mapreduce.output.fileoutputformat.compress=true                         mapreduce job 的结果输出是否使用压缩

mapreduce.job.reduces=10                                                reduce 个数
hive.exec.reducers.bytes.per.reducer=5120000000                         默认 1G，设为 5G，设置每个 reduce 的大小，hive 根据数据的总大小来猜测确定一个 reducer 的个数
```

其实本质上来说，减少 reduce 的数量就行，因为最终文件（如果有 reduce 阶段）个数取决于 reduce 个数。

---

## 二、数据倾斜问题

什么是数据倾斜？说白了，就是某个 task 被分配了过多的数据量，然后其他 task 都处理完了，都在等待这个 task 处理结束，本质上就是数据分布不均。

常见场景：

1. **join**：关联 key 存在一对多、发散情况，或者关联时脏数据、存在大量空值
2. **group by**：分组维度分布不均匀，某个值过多
3. **count distinct 语法**：依然是某个值过多的问题

原因：

- key 分布不均匀
- 源表数据问题、脏数据、一对多
- 业务建模时考虑不周
- SQL 语句写的不优雅、不健壮

### 2.1 Hive 的解决方式

参数调节：

```properties
hive.map.aggr=true                       map 端预聚合
hive.groupby.skewindata=true            数据倾斜负载均衡
hive.optimize.skewjoin=true             join 优化
```

> 更多的倾斜参数，可以官网参数文档搜索 `skew` 字样查询。

除了参数调节之外，SQL 语句才是首要调整的：

- **大小表 join**：小表进行 mapjoin。上古版 Hive 要加 hint，也就是 `select /*+ mapjoin(b) */ col from a left join b on a.x = b.x`，基本上现在默认启动该优化，可以设置 `hive.mapjoin.smalltable.filesize` 来决定表的大小
- **大表 join 大表**：关联 key 随机数拼接法、过滤空值；如果可以调研发散 key 的话，可以分步 join，最后再 union 结果
- **count distinct 语法改造**：不要用 `select col1, count(distinct col2) from a group by col1`，改用：

```sql
select col1, count(1)
from (select col1, col2 from a group by col1, col2) t
group by col1
```

### 2.2 SparkSQL 的解决方式

与 Hive 的方式较为类似：

- 对小表进行广播 **Broadcast Join**。SparkSQL 做了很多优化，底层运行时可以根据表实际文件大小来判断，当然也可以添加 hint 来强制 Broadcast Join，广播的大小受 `spark.sql.autoBroadcastJoinThreshold` 参数控制，默认 10M
- 大表 join 大表同 Hive 操作
- Spark 也有对应的 skew 参数，可以根据需要去官网查看
- 提高 shuffle 的并行度
- **两阶段聚合**：也就是局部聚合 + 全局聚合，先打随机前缀，然后第一次聚合，再将前缀去掉，第二次聚合
- SQL 语句同 Hive

> 总记：调优方法具体还是得要靠监控、调研、不断的实验。
