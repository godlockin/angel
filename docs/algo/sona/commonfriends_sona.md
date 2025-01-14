# CommonFriends

>Common Friends算法，顾名思义，旨在挖掘两个用户的共同好友数(Common Friends)；作为常见的图特征/指标，它常用来刻画用户之间的关系紧密程度，并且被广泛用于好友推荐，社区划分，熟人/陌生人分析等场景。

## 1. 算法介绍

Common Friends该算法的使用兼顾三种场景:

一、输入全量的关系链，计算已存在连边的共同好友数，可用于刻画关系紧密程度。

二、输入全量关系链和待计算共同好友的边表，计算指定连边的共同好友数，可用于连接预测或推理。

三、增量计算：当输入graph每隔一段时间有新增边，并需要例行化计算全量共同好友时，可在原计算结果的基础上做增量计算，适用于输入graph较大，而新增边相对较少，新加入的节点或边只影响graph中少部分边的共同好友个数的场景，相比每次进行全量计算，耗时显著降低。

## 2. 分布式实现

Common Friends的实现过程中，需要将顶点的邻接表存储在多个PS上。共同的好友的计算逻辑发生在worker上，此时需要从PS拉取两个顶点的邻接表计算交集，从而得到共同好友数。



## 3. 运行

### 算法IO参数
  - input： 输入，hdfs路径，无向图，不带权，每行表示一条边，srcId 分隔符 dstId，分隔符可以为空白符、tab或逗号等
  - extraInput: 输入，hdfs路径，数据格式要求同input，当extraInput和input路径一致时为算法的第一种使用场景，旨在计算全量边的共同好友数。当extraInput和input路径不同时，旨在计算给定连边的共同好友数，即算法的第二种使用场景。
  - output： 输出，hdfs路径，保存计算结果。每行表示srcId dstId 共同好友数。
  - sep: 分隔符，输入中每条边的起始顶点、目标顶点之间的分隔符: `tab`, `空格`等
### 算法参数
  - partitionNum： 数据分区数，worker端spark rdd的数据分区数量，一般设为spark executor个数乘以executor core数的3-4倍，
  - psPartitionNum： 参数服务器上模型的分区数量，最好是parameter server个数的整数倍，让每个ps承载的分区数量相等，让每个PS负载尽量均衡, 数据量大的话推荐500以上
  - batchSize： 向ps推送邻接表时的mini batch大小
  - pullBatchSize： 每个mini batch的大小
  - isCompressed：边是否压缩，1表示压缩的双向边
  - isIncremented：是否增量计算，设为True时需要填写增量边路径，同时input必须是已计算好的全量边结果
  - incEdgesPath：增量边路径
  - maxNodeId：graph中的最大节点id，当isIncremented设为True时需要填写
  - minNodeId：graph中的最小节点id，当isIncremented设为True时需要填写
  - maxComFriendsNum：最大的共同好友数，当某条边的共同好友个数小于等于该值时，输出共同好友数，否则输出-1
  - storageLevel：RDD存储级别，`DISK_ONLY`/`MEMORY_ONLY`/`MEMORY_AND_DISK`

### 资源参数

- ps个数和内存大小：ps.instance与ps.memory的乘积是ps总的配置内存。为了保证Angel不挂掉，需要配置ps上数据存储量大小两倍左右的内存。对于共同好友算法来说，ps上放置的是各顶点的一阶邻居，数据类型是(Long，Array[Long]),据此可以估算不同规模的Graph输入下需要配置的ps内存大小
- Spark的资源配置：num-executors与executor-memory的乘积是executors总的配置内存，最好能存下2倍的输入数据。 如果内存紧张，1倍也是可以接受的，但是相对会慢一点。 比如说100亿的边集大概有600G大小， 50G * 20 的配置是足够的。 在资源实在紧张的情况下， 尝试加大分区数目！

### 任务提交示例

```
input=hdfs://my-hdfs/data
extraInput=hdfs://my-hdfs/data
output=hdfs://my-hdfs/output

source ./spark-on-angel-env.sh
$SPARK_HOME/bin/spark-submit \
  --master yarn-cluster\
  --conf spark.ps.instances=1 \
  --conf spark.ps.cores=1 \
  --conf spark.ps.jars=$SONA_ANGEL_JARS \
  --conf spark.ps.memory=10g \
  --name "commonfriends angel" \
  --jars $SONA_SPARK_JARS  \
  --driver-memory 5g \
  --num-executors 1 \
  --executor-cores 4 \
  --executor-memory 10g \
  --class org.apache.spark.angel.examples.cluster.CommonFriendsExample \
  ../lib/spark-on-angel-examples-3.2.0.jar
  input:$input extraInput:$extraInput output:$output sep:tab storageLevel:MEMORY_ONLY useBalancePartition:true \
  partitionNum:4 psPartitionNum:1 batchSize:3000 pullBatchSize:1000 src:1 dst:2 mode:yarn-cluster
```



### 常见问题
  - 在差不多10min的时候，任务挂掉： 很可能的原因是angel申请不到资源！由于CommonFriends基于Spark On Angel开发，实际上涉及到Spark和Angel两个系统，它们的向Yarn申请资源是独立进行的。 在Spark任务拉起之后，由Spark向Yarn提交Angel的任务，如果不能在给定时间内申请到资源，就会报超时错误，任务挂掉！ 解决方案是： 1）确认资源池有足够的资源 2） 添加spakr conf: spark.hadoop.angel.am.appstate.timeout.ms=xxx 调大超时时间，默认值为600000，也就是10分钟
  - 如何估算我需要配置多少Angel资源： 为了保证Angel不挂掉，需要配置模型大小两倍左右的内存 另外，在可能的情况下，ps数目越小，数据传输的量会越小，但是单个ps的压力会越大，需要一定的权衡。
  - Spark的资源配置： 同样主要考虑内存问题，最好能存下2倍的输入数据。 如果内存紧张，1倍也是可以接受的，但是相对会慢一点。 比如说100亿的边集大概有600G大小， 50G * 20 的配置是足够的。 在资源实在紧张的情况下， 尝试加大分区数目！
