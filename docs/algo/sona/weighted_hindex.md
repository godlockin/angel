# H-Index
## 1. 算法介绍
Weighted HIndex算法是计算一个带权图中的节点h-index指数的算法。与HIndex不同的是，该算法考虑了边权重对节点指数的影响，具体可参考[Vital nodes identification in complex networks](https://www.sciencedirect.com/science/article/abs/pii/S0370157316301570)第51页。通常来说，h-index值越高，表明该节点的影响力越大，适用于社交网络中关键节点挖掘等场景。

## 2. 运行
#### 算法IO参数

- input：输入，hdfs路径，无向图，不带权。每行表示一条边： srcId 分隔符 dstId
- output: 输出，hdfs路径。每行表示一个顶点及其对应的hindex值：nodeId tab hindex值
- isWeighted：边是否带权，当为false时默认边权为1.0
- sep: 分隔符，输入中每条边的起始顶点、目标顶点之间的分隔符: `tab`, `空格`等

#### 算法参数

- partitionNum：数据分区数，spark rdd数据的分区数量
- psPartitionNum：参数服务器上模型的分区数量
- useBalancePartition：参数服务器对输入数据节点存储划分是否均衡分区，如果输入节点的索引不是均匀的话建议选择是
- storageLevel：RDD存储级别，`DISK_ONLY`/`MEMORY_ONLY`/`MEMORY_AND_DISK`

#### 资源参数

- ps个数和内存大小：ps.instance与ps.memory的乘积是ps总的配置内存。为了保证Angel不挂掉，需要配置ps上数据存储量大小两倍左右的内存。对于HIndex来说，ps上放置的是(nodeId,degree)的key-value结构的vector，数据类型是(Long,Int),据此可以估算不同规模的Graph输入下需要配置的ps内存大小
- Spark的资源配置：num-executors与executor-memory的乘积是executors总的配置内存，最好能存下2倍的输入数据。 如果内存紧张，1倍也是可以接受的，但是相对会慢一点。 比如说100亿的边集大概有600G大小， 50G * 20 的配置是足够的。 在资源实在紧张的情况下， 尝试加大分区数目！

#### 任务提交示例

```
input=hdfs://my-hdfs/data
output=hdfs://my-hdfs/output

source ./spark-on-angel-env.sh
$SPARK_HOME/bin/spark-submit \
  --master yarn-cluster\
  --conf spark.ps.instances=1 \
  --conf spark.ps.cores=1 \
  --conf spark.ps.jars=$SONA_ANGEL_JARS \
  --conf spark.ps.memory=10g \
  --name "whindex angel" \
  --jars $SONA_SPARK_JARS  \
  --driver-memory 5g \
  --num-executors 1 \
  --executor-cores 4 \
  --executor-memory 10g \
  --class org.apache.spark.angel.examples.graph.WeightedHIndexExample \
  ../lib/spark-on-angel-examples-3.2.0.jar
  input:$input output:$output sep:tab storageLevel:MEMORY_ONLY useBalancePartition:true \
  partitionNum:4 psPartitionNum:1
```

#### 常见问题
- 在差不多10min的时候，任务挂掉： 很可能的原因是angel申请不到资源！由于HIndex基于Spark On Angel开发，实际上涉及到Spark和Angel两个系统，在向Yarn申请资源时是独立进行的。 在Spark任务拉起之后，由Spark向Yarn提交Angel的任务，如果不能在给定时间内申请到资源，就会报超时错误，任务挂掉！ 解决方案是： 1）确认资源池有足够的资源 2） 添加spakr conf: spark.hadoop.angel.am.appstate.timeout.ms=xxx 调大超时时间，默认值为600000，也就是10分钟

