# 数据处理组件详解

## 目录
- [Spark核心概念](#spark核心概念)
- [Spark SQL与结构化数据](#spark-sql与结构化数据)
- [Spark Streaming流式处理](#spark-streaming流式处理)
- [Flink分布式计算引擎](#flink分布式计算引擎)
- [性能调优与最佳实践](#性能调优与最佳实践)

## Spark核心概念

### RDD（弹性分布式数据集）
RDD是Spark的基础抽象，代表一个不可变、可分区、里面的元素可并行计算的集合。

**RDD特性**：
- **分区**：数据被分割为多个分区，分布在集群节点
- **不可变**：RDD创建后不能修改，只能通过转换生成新RDD
- **容错性**：基于Lineage（血统）实现自动容错
- **惰性求值**：转换操作不会立即执行，直到遇到行动操作

**RDD操作类型**：
- **转换（Transformation）**：惰性操作，返回新RDD
  - map、flatMap、filter、mapPartitions
  - reduceByKey、groupByKey、join、cogroup
  - union、intersection、distinct、coalesce
- **行动（Action）**：触发任务执行
  - collect、count、reduce、take
  - saveAsTextFile、saveAsSequenceFile
  - foreach

**RDD示例**：
```scala
val conf = new SparkConf().setAppName("RDDExample").setMaster("local[*]")
val sc = new SparkContext(conf)

// 创建RDD
val data = sc.parallelize(1 to 10)
val textRDD = sc.textFile("hdfs://path/to/file")

// 转换操作
val filtered = data.filter(_ % 2 == 0)
val squared = filtered.map(x => x * x)
val pairs = squared.map(x => (x, 1))

// 行动操作
val result = pairs.collect()
val sum = pairs.reduce(_ + _._1)
pairs.saveAsTextFile("hdfs://output/path")
```

### Spark架构
**Driver Program**：
- 运行用户main函数，创建SparkContext
- 负责任务调度和DAG构建
- 维护RDD的血统关系

**Cluster Manager**：
- Standalone：Spark自带的资源管理器
- YARN：Hadoop的资源管理器
- Mesos：通用资源管理器
- Kubernetes：容器编排平台

**Executor**：
- 运行在Worker节点上的进程
- 负责运行Task并存储数据
- 通过心跳向Driver汇报状态

**Task**：
- 最小的执行单元，对应一个分区的数据处理
- 由Executor执行，结果返回给Driver

### DAG调度与Stage划分
**DAG（Directed Acyclic Graph）**：Spark根据RDD依赖关系构建的有向无环图

**Stage划分原则**：
- 宽依赖（Shuffle依赖）会触发Stage划分
- 窄依赖（如map、filter）在同一个Stage
- 每个Stage包含多个Task

**窄依赖 vs 宽依赖**：
- **窄依赖**：父RDD的每个分区最多被子RDD的一个分区使用
  - 优点：不需要Shuffle，流水线执行
  - 示例：map、filter、union
- **宽依赖**：父RDD的每个分区被子RDD的多个分区使用
  - 缺点：需要Shuffle，网络传输开销大
  - 示例：groupByKey、reduceByKey、join

### 内存管理
**存储级别**：
```scala
val rdd = sc.textFile("hdfs://path")
rdd.persist(StorageLevel.MEMORY_ONLY)           // 仅内存
rdd.persist(StorageLevel.MEMORY_AND_DISK)       // 内存+磁盘
rdd.persist(StorageLevel.MEMORY_ONLY_SER)       // 序列化到内存
rdd.persist(StorageLevel.DISK_ONLY)             // 仅磁盘
rdd.unpersist()                                  // 释放缓存
```

**内存模型**：
- **堆内内存**：JVM堆内存，通过SparkMemoryManager管理
- **堆外内存**：系统内存，通过Unsafe API操作，减少GC
- **存储内存**：缓存RDD数据
- **执行内存**：Shuffle、Join等操作使用

## Spark SQL与结构化数据

### DataFrame与Dataset
**DataFrame**：
- 结构化数据集合，类似数据库表
- 包含Schema（列名、数据类型）
- 支持SQL查询和DSL操作

**Dataset**：
- 类型安全的DataFrame，Scala/Java特有
- 编译时类型检查
- 综合了RDD和DataFrame的优势

**DataFrame示例**：
```scala
import org.apache.spark.sql.SparkSession

val spark = SparkSession.builder()
  .appName("SparkSQLExample")
  .master("local[*]")
  .getOrCreate()

// 读取数据
val df = spark.read.json("people.json")
val csvDF = spark.read.csv("data.csv")
val parquetDF = spark.read.parquet("data.parquet")

// 查看Schema
df.printSchema()

// DSL操作
df.select("name").show()
df.filter($"age" > 30).show()
df.groupBy("department").agg(avg("salary")).show()

// SQL查询
df.createOrReplaceTempView("people")
val result = spark.sql("SELECT name, age FROM people WHERE age > 30")
result.show()

// 写入数据
df.write.parquet("output.parquet")
df.write.mode("append").json("output.json")
```

### Catalyst优化器
**优化阶段**：
1. **未解析的逻辑计划**：解析SQL，检查语法
2. **逻辑计划**：生成逻辑执行计划
3. **优化的逻辑计划**：应用规则优化（谓词下推、列剪裁）
4. **物理计划**：选择最佳执行策略（Join算法、聚合方式）
5. **成本模型**：估算执行成本，选择最优计划

**常见优化规则**：
- **谓词下推**：将过滤条件尽可能下推到数据源
- **列剪裁**：只读取需要的列，减少IO
- **常量折叠**：编译时计算常量表达式
- **Join重排序**：选择最优Join顺序

### Hive集成
```scala
// 启用Hive支持
val spark = SparkSession.builder()
  .appName("HiveIntegration")
  .enableHiveSupport()
  .getOrCreate()

// 查询Hive表
val hiveDF = spark.sql("SELECT * FROM default.users WHERE age > 25")

// 创建Hive表
spark.sql("""
  CREATE TABLE IF NOT EXISTS employees (
    id INT,
    name STRING,
    salary DOUBLE
  )
  PARTITIONED BY (department STRING)
  STORED AS PARQUET
""")

// 写入Hive表
df.write.mode("overwrite").insertInto("employees")
```

### 自定义函数UDF
```scala
// 定义UDF
val upperUDF = udf((s: String) => s.toUpperCase)

// 注册UDF
spark.udf.register("upper", upperUDF)

// 使用UDF
spark.sql("SELECT upper(name) FROM people").show()

// Spark 3.x强类型UDF（推荐）
import org.apache.spark.sql.expressions.UserDefinedFunction
val multiply: UserDefinedFunction = udf((x: Int, y: Int) => x * y)

// DataFrame API
df.withColumn("result", multiply(col("a"), col("b")))
```

## Spark Streaming流式处理

### DStream（离散流）
**DStream概念**：Spark Streaming的基本抽象，代表连续的数据流，由一系列连续的RDD组成

**批处理间隔**：数据流被分割成多个批次，每个批处理间隔生成一个RDD

**DStream示例**：
```scala
import org.apache.spark.streaming.{Seconds, StreamingContext}
import org.apache.spark.streaming.StreamingContext._

val ssc = new StreamingContext(spark.sparkContext, Seconds(5))

// 创建DStream
val lines = ssc.socketTextStream("localhost", 9999)

// 转换操作
val words = lines.flatMap(_.split(" "))
val pairs = words.map(word => (word, 1))
val wordCounts = pairs.reduceByKey(_ + _)

// 输出操作
wordCounts.print()

// 启动流处理
ssc.start()
ssc.awaitTermination()
```

### 操作类型
- **转换操作**：类似RDD转换
  - map、flatMap、filter、reduceByKey
  - transform：将DStream转为RDD进行自定义操作
  - updateStateByKey：维护状态，计算累计值
  - window：窗口操作（滑动窗口、滚动窗口）

- **输出操作**：触发计算
  - print、saveAsTextFiles、foreachRDD

**窗口操作示例**：
```scala
// 滑动窗口：窗口长度30秒，滑动间隔10秒
val windowedCounts = pairs.reduceByKeyAndWindow(
  (a: Int, b: Int) => a + b,
  Seconds(30),
  Seconds(10)
)

// 窗口去重（可选）
val deduped = windowedCounts.transform(rdd => rdd.reduceByKey(_ + _))
```

### 检查点（Checkpoint）
**用途**：容错和状态恢复

**配置检查点**：
```scala
ssc.checkpoint("hdfs://checkpoint-dir")

// 需要检查点的操作
updateStateByKey
window (逆序窗口)
```

### Structured Streaming（推荐）
Structured Streaming是基于Spark SQL的流处理引擎，提供更强大的功能。

**核心思想**：
- 将流数据视为无界表
- 持续追加数据到表
- 支持批流统一API

**Structured Streaming示例**：
```scala
val spark = SparkSession.builder()
  .appName("StructuredStreaming")
  .master("local[*]")
  .getOrCreate()

import spark.implicits._

// 读取流数据
val lines = spark.readStream
  .format("socket")
  .option("host", "localhost")
  .option("port", 9999)
  .load()

// 转换操作
val wordCounts = lines.as[String]
  .flatMap(_.split(" "))
  .groupBy("value")
  .count()

// 输出模式
val query = wordCounts.writeStream
  .outputMode("complete")  // complete、append、update
  .format("console")
  .start()

query.awaitTermination()
```

## Flink分布式计算引擎

### Flink架构
**JobManager**：
- 接收JobGraph，调度任务
- 协调Checkpoint和故障恢复
- 管理任务槽位（Slot）

**TaskManager**：
- 执行任务，管理Task Slot
- 向JobManager汇报状态
- 管理数据交换

**任务槽位（Slot）**：
- TaskManager的资源抽象
- 每个Slot隔离资源（内存、CPU）
- 一个Task可以共享Slot（链式优化）

### DataStream API
**DataStream**：Flink流处理的核心API

**基础操作**：
```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.datastream.DataStream;

// 创建执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

// 并行度设置
env.setParallelism(4);

// 创建DataStream
DataStream<String> text = env.socketTextStream("localhost", 9999);

// 转换操作
DataStream<Tuple2<String, Integer>> counts = text
    .flatMap(new Tokenizer())
    .keyBy(0)
    .sum(1);

// 执行
env.execute("Word Count");
```

**KeyedStream**：
- 根据Key分区，确保相同Key的数据去同一个分区
- 支持状态管理和窗口操作

```java
// KeyBy分区
KeyedStream<Tuple2<String, Integer>, Tuple> keyed = counts.keyBy(0);

// 窗口操作
DataStream<Tuple2<String, Integer>> windowed = keyed
    .window(TumblingProcessingTimeWindows.of(Time.seconds(5)))
    .sum(1);
```

### 窗口操作
**窗口类型**：
- **滚动窗口（Tumbling Window）**：窗口不重叠
- **滑动窗口（Sliding Window）**：窗口有重叠
- **会话窗口（Session Window）**：根据数据间隔动态划分

**时间语义**：
- **Processing Time**：处理时间（机器时间），无序但低延迟
- **Event Time**：事件时间（数据时间），有序但需要Watermark
- **Ingestion Time**：摄入时间（进入Flink时间）

**Watermark**：
- 用于处理乱序事件
- 标记事件时间推进
- 触发窗口计算

```java
// 定义Watermark策略
WatermarkStrategy<Tuple2<String, Long>> watermarkStrategy = WatermarkStrategy
    .<Tuple2<String, Long>>forBoundedOutOfOrderness(Duration.ofSeconds(5))
    .withTimestampAssigner((event, timestamp) -> event.f1);

// 应用Watermark
DataStream<Tuple2<String, Long>> withWatermark = socketStream
    .assignTimestampsAndWatermarks(watermarkStrategy);
```

### 状态管理
**状态类型**：
- **Keyed State**：绑定到Key的状态
- **Operator State**：绑定到算子的状态

**状态后端**：
- **MemoryStateBackend**：内存状态，适合小状态和测试
- **FsStateBackend**：文件系统状态，适合大状态
- **RocksDBStateBackend**：RocksDB存储，适合超大状态

```java
// 配置状态后端
env.setStateBackend(new RocksDBStateBackend("hdfs://checkpoint-dir"));

// ValueState示例
public class MyMapper extends KeyedProcessFunction<String, Tuple2<String, Integer>, Tuple2<String, Integer>> {

    private ValueState<Integer> count;

    @Override
    public void open(Configuration parameters) {
        ValueStateDescriptor<Integer> descriptor =
            new ValueStateDescriptor<>("count", Integer.class);
        count = getRuntimeContext().getState(descriptor);
    }

    @Override
    public void processElement(Tuple2<String, Integer> value, Context ctx, Collector<Tuple2<String, Integer>> out) throws Exception {
        Integer currentCount = count.value();
        if (currentCount == null) {
            currentCount = 0;
        }
        count.update(currentCount + value.f1);
        out.collect(new Tuple2<>(value.f0, count.value()));
    }
}
```

### Checkpoint与容错
**Checkpoint机制**：
- 定期异步保存全局一致性快照
- 基于Chandy-Lamport算法
- 任务失败时从最近Checkpoint恢复

**配置Checkpoint**：
```java
// 开启Checkpoint，间隔1分钟
env.enableCheckpointing(60000);

// Checkpoint配置
CheckpointConfig checkpointConfig = env.getCheckpointConfig();
checkpointConfig.setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);
checkpointConfig.setMinPauseBetweenCheckpoints(500);
checkpointConfig.setCheckpointTimeout(60000);
checkpointConfig.setMaxConcurrentCheckpoints(1);
checkpointConfig.enableExternalizedCheckpoints(CheckpointConfig.ExternalizedCheckpointCleanup.RETAIN_ON_CANCELLATION);
```

## 性能调优与最佳实践

### Spark性能调优

**并行度调优**：
```scala
// 设置并行度
val rdd = sc.parallelize(data, 100)
val df = spark.read.parquet("data").repartition(200)

// 调整Shuffle分区数
spark.conf.set("spark.sql.shuffle.partitions", "200")
```

**内存调优**：
```bash
# Driver内存
--driver-memory 4g

# Executor内存
--executor-memory 8g

# 堆外内存
--conf spark.memory.offHeap.enabled=true
--conf spark.memory.offHeap.size=4g
```

**数据倾斜处理**：
```scala
// 采样分析倾斜Key
val sampled = df.sample(false, 0.01).groupBy("key").count().sort(desc("count"))

// 增加并行度
df.repartition(1000, $"key")

// Broadcast Join（小表）
val smallDF = spark.read.parquet("small_table")
val broadcastDF = broadcast(smallDF)
df.join(broadcastDF, "key")

// 两阶段聚合（解决groupByKey倾斜）
val rdd = pairs.mapPartitions { iter =>
  iter.groupBy(_._1).mapValues(_.map(_._2).sum).iterator
}
```

**序列化调优**：
```scala
// Kryo序列化（比Java序列化快10倍）
spark.conf.set("spark.serializer", "org.apache.spark.serializer.KryoSerializer")
spark.conf.set("spark.kryo.registrationRequired", "true")
KryoRegistrar.registerClasses(Array(
  classOf[MyClass],
  classOf[MyCaseClass]
))
```

### Flink性能调优

**并行度设置**：
```java
// 全局并行度
env.setParallelism(4);

// 单个算子并行度
stream.keyBy(0).sum(1).setParallelism(2);
```

**任务槽位优化**：
```bash
# TaskManager内存
taskmanager.memory.process.size: 4096m

# Slot数量
taskmanager.numberOfTaskSlots: 4

# 每个Slot内存
taskmanager.memory.task.heap.size: 2048m
```

**Checkpoint优化**：
```java
// 异步Checkpoint
env.getCheckpointConfig().setCheckpointingMode(CheckpointingMode.EXACTLY_ONCE);

// 减少Checkpoint间隔
env.enableCheckpointing(30000);

// 增加Checkpoint超时
env.getCheckpointConfig().setCheckpointTimeout(600000);
```

**Backpressure处理**：
- 调整网络缓冲区：`taskmanager.network.memory.fraction`
- 优化算子逻辑：避免阻塞操作
- 增加并行度：提高吞吐量

### 通用最佳实践

**数据格式选择**：
- **Parquet**：列式存储，适合OLAP查询
- **ORC**：列式存储，Hive优化
- **Avro**：行式存储，适合OLTP

**分区策略**：
- 按时间分区：`dt=2024-01-01/hour=10`
- 按业务字段分区：避免分区过多（<1000）
- 分桶：用于Join优化

**监控指标**：
- **Spark**：UI、Metrics、日志
- **Flink**：Web Dashboard、Metrics、Checkpoint状态

**常见问题排查**：
1. OOM：增加内存、减少数据量、优化算法
2. 数据倾斜：采样分析、增加并行度、两阶段聚合
3. 任务慢：查看执行计划、优化Shuffle、调整并行度
4. Checkpoint失败：增加超时、调整频率、优化状态大小
