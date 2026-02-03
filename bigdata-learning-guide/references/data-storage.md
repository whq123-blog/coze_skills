# 数据存储组件详解

## 目录
- [HBase分布式数据库](#hbase分布式数据库)
- [Hive数据仓库](#hive数据仓库)
- [ClickHouse列式数据库](#clickhouse列式数据库)
- [Kafka消息队列](#kafka消息队列)
- [选型与最佳实践](#选型与最佳实践)

## HBase分布式数据库

### 核心概念
HBase是高可靠性、高性能、列式可伸缩的NoSQL数据库，基于Hadoop HDFS构建。

**数据模型**：
- **Table**：表，由行和列组成
- **Row**：行，通过RowKey唯一标识
- **Column Family**：列族，一组列的集合，物理存储在一起
- **Column**：列，由列族和限定符（Qualifier）标识
- **Timestamp**：时间戳，版本控制
- **Cell**：单元格，由{RowKey, Column Family, Column, Timestamp}唯一确定

**表结构示例**：
```
Table: users
+----------------+-------------------+-------------------+
| RowKey         | info              | contact           |
|                | name | age        | email | phone     |
+----------------+-------------------+-------------------+
| user_001       | Tom  | 28         | tom@... | 138...   |
| user_002       | Jack | 35         | jack@... | 139...  |
+----------------+-------------------+-------------------+
```

### 架构组件
**Master**：
- 管理表的DDL操作（创建、删除、修改）
- 负责Region的分配和负载均衡
- 监控RegionServer状态

**RegionServer**：
- 管理多个Region
- 处理读写请求
- 负责Region的Split和Compact

**Region**：
- 表的分区，按RowKey范围划分
- 每个Region由一个RegionServer管理
- Region大小超过阈值自动Split

**ZooKeeper**：
- 集群协调服务
- Master选举
- RegionServer注册和监控

### RowKey设计原则
RowKey是HBase最重要的设计，直接影响查询性能。

**设计原则**：
1. **唯一性**：RowKey必须唯一标识一条记录
2. **长度控制**：建议16-100字节，避免过长
3. **散列性**：避免热点问题，均匀分布数据
4. **排序性**：利用RowKey有序性，优化范围查询

**常见RowKey设计方案**：

1. **加盐（Salt）**：防止热点
   ```
   RowKey = hash(userId) + userId + timestamp
   ```

2. **哈希**：均匀分布
   ```
   RowKey = MD5(userId).substring(0, 4) + userId
   ```

3. **反转**：时间序列数据
   ```
   RowKey = reverse(timestamp) + userId
   ```

4. **组合键**：多维度查询
   ```
   RowKey = userId + timestamp + deviceId
   ```

**示例**：
```java
// 创建表
HBaseAdmin admin = connection.getAdmin();
TableName tableName = TableName.valueOf("users");
HTableDescriptor tableDescriptor = new HTableDescriptor(tableName);

// 创建列族
HColumnDescriptor infoFamily = new HColumnDescriptor("info");
infoFamily.setVersions(1, 3);  // 最小1个版本，最大3个版本
infoFamily.setCompressionType(Compression.Algorithm.SNAPPY);
tableDescriptor.addFamily(infoFamily);

// 创建表
admin.createTable(tableDescriptor);

// 插入数据
Table table = connection.getTable(tableName);
Put put = new Put(Bytes.toBytes("user_001"));
put.addColumn(Bytes.toBytes("info"), Bytes.toBytes("name"), Bytes.toBytes("Tom"));
put.addColumn(Bytes.toBytes("info"), Bytes.toBytes("age"), Bytes.toBytes(28));
table.put(put);

// 查询数据
Get get = new Get(Bytes.toBytes("user_001"));
Result result = table.get(get);
String name = Bytes.toString(result.getValue(Bytes.toBytes("info"), Bytes.toBytes("name")));

// 范围查询
Scan scan = new Scan();
scan.setRowPrefixFilter(Bytes.toBytes("user_"));  // 前缀查询
scan.setStartRow(Bytes.toBytes("user_001"));
scan.setStopRow(Bytes.toBytes("user_010"));
ResultScanner scanner = table.getScanner(scan);
```

### 读写流程
**写流程**：
1. 客户端向RegionServer发送写请求
2. 数据先写入WAL（Write-Ahead Log）
3. 数据写入MemStore（内存）
4. 返回成功
5. MemStore满后 Flush到HFile（磁盘）

**读流程**：
1. 客户端向RegionServer发送读请求
2. 查找BlockCache（缓存）
3. 查找MemStore（内存）
4. 查找HFile（磁盘）
5. 合并多个版本数据
6. 返回结果

**优化策略**：
- **BlockCache**：LRU缓存，加速读操作
- **BloomFilter**：布隆过滤器，快速判断数据是否存在
- **Compression**：数据压缩，减少磁盘IO

### 性能优化
**列族配置**：
```xml
<property>
  <name>hbase.regionserver.handler.count</name>
  <value>50</value>
</property>
<property>
  <name>hbase.hstore.blockingStoreFiles</name>
  <value>15</value>
</property>
<property>
  <name>hbase.regionserver.global.memstore.lowerLimit</name>
  <value>0.35</value>
</property>
```

**常见问题解决**：
1. **热点问题**：RowKey加Hash或Salt
2. **写入慢**：增加MemStore、调整Flush频率
3. **查询慢**：启用BlockCache、BloomFilter，优化RowKey
4. **RegionServer OOM**：调整堆内存、减少MemStore

## Hive数据仓库

### 核心概念
Hive是基于Hadoop的数据仓库工具，提供类似SQL的查询语言HiveQL，用于结构化数据分析。

**Hive架构**：
- **用户接口**：CLI、JDBC、WebUI
- **元数据存储**：Metastore（默认使用Derby，生产环境使用MySQL）
- **驱动器**：编译器、优化器、执行器
- **执行引擎**：MapReduce、Tez、Spark

**Hive vs 传统数据库**：
| 特性 | Hive | 传统数据库 |
|------|------|-----------|
| 数据规模 | TB/PB级 | GB级 |
| 延迟 | 高延迟（秒级） | 低延迟（毫秒级） |
| 事务 | 支持有限 | 完整ACID |
| 更新/删除 | 支持较差 | 原生支持 |
| 索引 | 支持有限 | 完整索引 |

### 数据模型
**内部表（Managed Table）**：
- Hive管理数据存储位置
- 删除表时，同时删除数据
- 默认存储在`/user/hive/warehouse`

**外部表（External Table）**：
- 用户指定数据存储位置
- 删除表时，只删除元数据，不删除数据
- 适合多工具共享数据

**分区表**：
- 按字段分区，提高查询性能
- 常用分区字段：日期、地区、业务类型
- 分区字段不是表的实际字段

**分桶表**：
- 将数据按哈希值分散到多个桶
- 提高Join性能
- 分桶字段必须是表的列

### HiveQL语法
**DDL操作**：
```sql
-- 创建内部表
CREATE TABLE users (
    id INT,
    name STRING,
    age INT,
    salary DOUBLE
)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY ','
STORED AS TEXTFILE;

-- 创建外部表
CREATE EXTERNAL TABLE logs (
    log_time STRING,
    level STRING,
    message STRING
)
PARTITIONED BY (dt STRING, hour STRING)
ROW FORMAT DELIMITED
FIELDS TERMINATED BY '\t'
STORED AS TEXTFILE
LOCATION '/user/hive/warehouse/logs';

-- 创建分区表
ALTER TABLE logs ADD PARTITION (dt='2024-01-01', hour='00')
LOCATION '/user/hive/warehouse/logs/2024-01-01/00';

-- 创建分桶表
CREATE TABLE user_buckets (
    id INT,
    name STRING
)
CLUSTERED BY (id) INTO 32 BUCKETS;

-- 创建ORC格式表（列式存储）
CREATE TABLE users_orc (
    id INT,
    name STRING,
    salary DOUBLE
)
STORED AS ORC
TBLPROPERTIES ("orc.compress"="SNAPPY");
```

**DML操作**：
```sql
-- 插入数据
INSERT INTO TABLE users VALUES (1, 'Tom', 28, 10000.0);

-- 从查询插入
INSERT OVERWRITE TABLE users_orc
SELECT id, name, salary FROM users;

-- 动态分区插入
SET hive.exec.dynamic.partition.mode=nonstrict;
INSERT OVERWRITE TABLE logs PARTITION (dt, hour)
SELECT log_time, level, message, dt, hour FROM temp_logs;

-- 导入数据
LOAD DATA LOCAL INPATH '/path/to/users.csv' INTO TABLE users;

-- 更新和删除（需要启用ACID）
SET hive.support.concurrency=true;
SET hive.txn.manager=org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;
SET hive.enforce.bucketing=true;
SET hive.exec.dynamic.partition.mode=nonstrict;

UPDATE users SET salary = salary * 1.1 WHERE age > 30;
DELETE FROM users WHERE id = 1;
```

**DQL查询**：
```sql
-- 基础查询
SELECT name, salary FROM users WHERE age > 30;

-- 聚合查询
SELECT department, AVG(salary) as avg_salary
FROM users
GROUP BY department
HAVING AVG(salary) > 10000;

-- Join查询
SELECT u.name, o.order_id
FROM users u
JOIN orders o ON u.id = o.user_id;

-- 窗口函数
SELECT
    name,
    salary,
    RANK() OVER (ORDER BY salary DESC) as salary_rank,
    SUM(salary) OVER (PARTITION BY department) as dept_total
FROM users;

-- 子查询
SELECT * FROM users
WHERE salary > (SELECT AVG(salary) FROM users);
```

### 性能优化
**存储格式选择**：
- **TextFile**：文本格式，便于查看，性能差
- **ORC**：列式存储，压缩比高，性能好（推荐）
- **Parquet**：列式存储，与Spark生态兼容
- **SequenceFile**：二进制格式，适合大量小文件

**压缩格式**：
```xml
<!-- 开启压缩 -->
<property>
  <name>hive.exec.compress.output</name>
  <value>true</value>
</property>
<property>
  <name>mapreduce.output.fileoutputformat.compress.codec</name>
  <value>org.apache.hadoop.io.compress.SnappyCodec</value>
</property>
```

**执行引擎选择**：
```xml
<!-- 使用Spark引擎 -->
<property>
  <name>hive.execution.engine</name>
  <value>spark</value>
</property>

<!-- 使用Tez引擎 -->
<property>
  <name>hive.execution.engine</name>
  <value>tez</value>
</property>
```

**Join优化**：
- **Map Join**：小表加载到内存，大表流式处理
  ```sql
  SET hive.auto.convert.join=true;
  SET hive.auto.convert.join.noconditionaltask=true;
  SET hive.auto.convert.join.noconditionaltask.size=10000000;
  ```

- **SMB Join**：分桶表Join，减少Shuffle
  ```sql
  SET hive.optimize.bucketmapjoin=true;
  SET hive.optimize.bucketmapjoin.sortedmerge=true;
  SET hive.input.format=org.apache.hadoop.hive.ql.io.BucketizedHiveInputFormat;
  ```

**查询优化**：
```sql
-- 开启谓词下推
SET hive.optimize.ppd=true;

-- 开启列剪裁
SET hive.optimize.cp=true;

-- 并行执行
SET hive.exec.parallel=true;
SET hive.exec.parallel.thread.number=16;

-- 向量化查询
SET hive.vectorized.execution.enabled=true;
```

### UDF开发
**UDF（User Defined Function）**：
```java
public class UpperUDF extends UDF {
    public String evaluate(String str) {
        if (str == null) return null;
        return str.toUpperCase();
    }
}

// 编译打包
// mvn package

// 添加JAR
ADD JAR /path/to/udf.jar;

// 创建临时函数
CREATE TEMPORARY FUNCTION upper_udf AS 'com.example.UpperUDF';

// 使用UDF
SELECT upper_udf(name) FROM users;
```

**UDAF（聚合函数）**：
```java
public class SumUDAF extends AbstractGenericUDAFResolver {
    @Override
    public AbstractGenericUDAFResolver getResolver(GenericUDAFParameters parameters) {
        return new GenericUDAFSum();
    }
}
```

## ClickHouse列式数据库

### 核心概念
ClickHouse是面向列的OLAP数据库，专为快速分析大规模数据设计。

**核心特性**：
- **列式存储**：按列存储，适合分析查询
- **向量化执行**：批量处理，提高性能
- **压缩率高**：列式压缩，节省存储
- **支持SQL**：兼容大部分SQL语法
- **实时写入**：支持高并发实时插入

**数据引擎**：
- **MergeTree**：基础引擎，支持索引
- **ReplicatedMergeTree**：副本引擎，高可用
- **SummingMergeTree**：预聚合引擎
- **AggregatingMergeTree**：聚合引擎
- **CollapsingMergeTree**：折叠引擎，支持更新

### 表结构设计
**MergeTree表**：
```sql
CREATE TABLE events (
    date Date,
    event_time DateTime,
    user_id UInt32,
    event_type String,
    event_data String
)
ENGINE = MergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, event_time)
SETTINGS index_granularity = 8192;
```

**ReplicatedMergeTree表（副本）**：
```sql
CREATE TABLE events_replica ON CLUSTER '{cluster}' (
    date Date,
    event_time DateTime,
    user_id UInt32,
    event_type String
)
ENGINE = ReplicatedMergeTree('/clickhouse/tables/{shard}/events', '{replica}')
PARTITION BY toYYYYMM(date)
ORDER BY (user_id, event_time);
```

**SummingMergeTree表（预聚合）**：
```sql
CREATE TABLE daily_sales (
    date Date,
    product_id UInt32,
    sales_amount Decimal(18, 2)
)
ENGINE = SummingMergeTree(sales_amount)
PARTITION BY toYYYYMM(date)
ORDER BY (date, product_id);
```

### 数据导入
**从文件导入**：
```sql
-- 导入CSV
INSERT INTO events FORMAT CSVWithNames
Date,Event_time,User_id,Event_type
2024-01-01,2024-01-01 10:00:00,1001,login
2024-01-01,2024-01-01 10:01:00,1002,click
...

-- 导入JSON
INSERT INTO events FORMAT JSONEachRow
{"date": "2024-01-01", "event_time": "2024-01-01 10:00:00", "user_id": 1001, "event_type": "login"}
{"date": "2024-01-01", "event_time": "2024-01-01 10:01:00", "user_id": 1002, "event_type": "click"}
```

**从MySQL同步**：
```sql
-- 创建MySQL表引擎
CREATE TABLE mysql_source (
    id UInt32,
    name String,
    value Float64
)
ENGINE = MySQL('mysql-host:3306', 'database', 'table', 'user', 'password');

-- 查询MySQL表
SELECT * FROM mysql_source;

-- 导入到ClickHouse表
INSERT INTO clickhouse_table
SELECT * FROM mysql_source;
```

### 查询优化
**索引利用**：
```sql
-- ORDER BY子句定义主键，查询时尽量使用主键
-- 高效查询
SELECT * FROM events WHERE user_id = 1001 AND event_time >= '2024-01-01';

-- 低效查询（未利用索引）
SELECT * FROM events WHERE event_type = 'login';
```

**聚合优化**：
```sql
-- 使用MATERIALIZED VIEW加速聚合
CREATE MATERIALIZED VIEW events_mv
ENGINE = SummingMergeTree()
PARTITION BY toYYYYMM(date)
ORDER BY (date, event_type)
AS SELECT
    date,
    event_type,
    count() as event_count
FROM events
GROUP BY date, event_type;

-- 查询物化视图
SELECT * FROM events_mv WHERE date = '2024-01-01';
```

**分区裁剪**：
```sql
-- 按日期分区，查询时指定日期范围
SELECT * FROM events WHERE date BETWEEN '2024-01-01' AND '2024-01-31';
```

## Kafka消息队列

### 核心概念
Kafka是分布式发布-订阅消息系统，用于处理实时数据流。

**核心概念**：
- **Broker**：Kafka服务器节点
- **Topic**：消息类别
- **Partition**：Topic的分区，提高并发和可扩展性
- **Offset**：消息在分区中的位置
- **Producer**：消息生产者
- **Consumer**：消息消费者
- **Consumer Group**：消费者组，多个消费者协同消费

**消息模型**：
- 发布-订阅模型：Producer发送消息到Topic，Consumer订阅Topic消费消息
- 分区模型：每个分区有序，全局无序
- 消费者组：每个分区只能被组内一个消费者消费

### 基本操作
**创建Topic**：
```bash
# 创建Topic
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic events \
  --partitions 3 \
  --replication-factor 2

# 查看Topic列表
kafka-topics.sh --list --bootstrap-server localhost:9092

# 查看Topic详情
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic events
```

**生产者代码**：
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("key.serializer", "org.apache.kafka.common.serialization.StringSerializer");
props.put("value.serializer", "org.apache.kafka.common.serialization.StringSerializer");

Producer<String, String> producer = new KafkaProducer<>(props);

for (int i = 0; i < 100; i++) {
    ProducerRecord<String, String> record =
        new ProducerRecord<>("events", "key" + i, "message" + i);
    producer.send(record);
}

producer.close();
```

**消费者代码**：
```java
Properties props = new Properties();
props.put("bootstrap.servers", "localhost:9092");
props.put("group.id", "test-group");
props.put("key.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");
props.put("value.deserializer", "org.apache.kafka.common.serialization.StringDeserializer");

KafkaConsumer<String, String> consumer = new KafkaConsumer<>(props);
consumer.subscribe(Arrays.asList("events"));

while (true) {
    ConsumerRecords<String, String> records = consumer.poll(Duration.ofMillis(100));
    for (ConsumerRecord<String, String> record : records) {
        System.out.printf("offset = %d, key = %s, value = %s%n",
            record.offset(), record.key(), record.value());
    }
}
```

### 消息可靠性
**生产者确认**：
```java
// 等待所有副本确认
props.put("acks", "all");

// 启用幂等性
props.put("enable.idempotence", true);

// 重试配置
props.put("retries", 3);
props.put("max.in.flight.requests.per.connection", 5);
```

**消费者提交**：
```java
// 自动提交（默认）
props.put("enable.auto.commit", "true");
props.put("auto.commit.interval.ms", "1000");

// 手动提交（推荐）
props.put("enable.auto.commit", "false");

consumer.commitSync();  // 同步提交
consumer.commitAsync();  // 异步提交
```

**消息不丢失**：
- **生产者**：启用`acks=all`，开启幂等性
- **Broker**：配置`replication.factor > 1`，`min.insync.replicas > 1`
- **消费者**：手动提交Offset，处理完成后再提交

### 消息重复与顺序
**消息去重**：
- 幂等性生产者（单Partition）
- 消费者端去重（基于唯一ID）

```java
// 消费者去重示例
Set<String> processedIds = new HashSet<>();
for (ConsumerRecord<String, String> record : records) {
    String messageId = record.key();
    if (!processedIds.contains(messageId)) {
        // 处理消息
        processMessage(record.value());
        processedIds.add(messageId);
    }
}
```

**消息顺序**：
- 单Partition内有序
- 多Partition间无序
- 如需全局有序，使用单Partition或自定义分区器

## 选型与最佳实践

### 存储组件对比
| 组件 | 适用场景 | 优点 | 缺点 |
|------|---------|------|------|
| **HBase** | 海量KV数据、实时读写 | 高扩展性、实时查询 | 不适合复杂查询、RowKey设计复杂 |
| **Hive** | 离线数据分析、数据仓库 | SQL友好、批处理 | 高延迟、不适合实时查询 |
| **ClickHouse** | OLAP分析、实时报表 | 查询快、压缩率高 | 不适合高并发更新、事务支持弱 |
| **Kafka** | 实时数据流、消息队列 | 高吞吐、持久化 | 不适合复杂查询 |

### 选型建议
**写入密集型**：
- 实时写入：Kafka + HBase
- 批量写入：Hive（ORC格式）

**查询密集型**：
- 实时查询：HBase
- 复杂分析：Hive + Spark
- OLAP报表：ClickHouse

**场景组合**：
- Lambda架构：Kafka + Spark Streaming + HBase + Hive
- Kappa架构：Kafka + Flink + HBase
- 实时数仓：Kafka + Flink + ClickHouse

### 性能调优总结
**HBase**：
- RowKey设计：避免热点，均匀分布
- 列族配置：减少列族数量，启用压缩
- Region划分：控制Region大小（10-20GB）

**Hive**：
- 存储格式：使用ORC/Parquet
- 压缩格式：Snappy/ZSTD
- Join优化：使用Map Join、SMB Join

**ClickHouse**：
- 索引利用：ORDER BY字段合理设计
- 分区策略：按时间或维度分区
- 物化视图：预聚合常用查询

**Kafka**：
- 分区设计：根据吞吐量设置分区数
- 副本配置：平衡可靠性和性能
- 批量发送：提高吞吐量
