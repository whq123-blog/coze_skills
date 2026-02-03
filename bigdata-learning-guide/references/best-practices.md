# 最佳实践与架构设计

## 目录
- [数仓分层设计](#数仓分层设计)
- [维度建模方法](#维度建模方法)
- [常见架构模式](#常见架构模式)
- [性能优化策略](#性能优化策略)
- [监控与运维](#监控与运维)

## 数仓分层设计

### 分层原则
数仓分层是数据仓库设计的核心，通过分层实现数据治理、降低耦合、提高复用性。

**分层价值**：
- **清晰结构**：数据流向清晰，便于维护
- **数据复用**：公共数据可被多个应用共享
- **降低耦合**：各层独立，变更影响范围小
- **性能优化**：按需查询，减少全表扫描
- **数据治理**：统一口径，避免数据孤岛

### 标准分层模型

**ODS层（Operational Data Store - 原始数据层）**
- **作用**：存储原始数据，与源系统保持一致
- **数据特征**：不做任何清洗、转换、加工
- **保留周期**：通常保留3-6个月，部分数据长期保留
- **存储格式**：原始格式（文本、JSON等）
- **示例**：MySQL Binlog、日志文件、接口原始数据

**DWD层（Data Warehouse Detail - 明细数据层）**
- **作用**：对ODS层数据进行清洗、规范化
- **数据特征**：去重、标准化、统一口径
- **保留周期**：长期保留，历史数据归档
- **存储格式**：ORC/Parquet，启用压缩
- **示例**：用户行为日志、订单明细、商品信息

**DWS层（Data Warehouse Summary - 汇总数据层）**
- **作用**：基于DWD层进行轻度汇总，面向业务分析
- **数据特征**：按维度汇总，生成指标
- **保留周期**：根据业务需求，通常6-12个月
- **存储格式**：ORC/Parquet，分区存储
- **示例**：日活跃用户数、日订单总额、商品日销量

**ADS层（Application Data Service - 应用数据层）**
- **作用**：面向具体业务应用，直接使用
- **数据特征**：高度汇总，多维度
- **保留周期**：根据应用需求，通常1-3个月
- **存储格式**：ORC/Parquet或MySQL
- **示例**：BI报表、实时看板、用户画像标签

### 分层示例

**电商数仓分层**：
```
ODS层：
- ods_user_log（用户行为日志原始数据）
- ods_order_detail（订单明细原始数据）
- ods_product_info（商品信息原始数据）

DWD层：
- dwd_user_action_log（清洗后的用户行为日志）
- dwd_order_detail（清洗后的订单明细）
- dwd_product_info（清洗后的商品信息）

DWS层：
- dws_user_daily（用户日活统计）
- dws_order_daily（订单日汇总）
- dws_product_daily（商品日销量统计）

ADS层：
- ads_user_analysis（用户分析报表）
- ads_order_report（订单报表）
- ads_product_ranking（商品排行榜）
```

**分层SQL示例**：
```sql
-- DWD层：用户行为日志清洗
INSERT OVERWRITE TABLE dwd_user_action_log
SELECT
    user_id,
    action_type,
    product_id,
    action_time,
    device_type,
    FROM_UNIXTIME(cast(ts as BIGINT)) as event_time
FROM ods_user_log
WHERE dt = '${dt}'
  AND user_id IS NOT NULL
  AND action_type IN ('view', 'click', 'cart', 'purchase');

-- DWS层：用户日活统计
INSERT OVERWRITE TABLE dws_user_daily
SELECT
    user_id,
    COUNT(DISTINCT action_time) as action_count,
    COUNT(DISTINCT product_id) as product_count,
    COUNT(DISTINCT CASE WHEN action_type = 'purchase' THEN product_id END) as purchase_count
FROM dwd_user_action_log
WHERE dt = '${dt}'
GROUP BY user_id;

-- ADS层：用户分析报表
INSERT OVERWRITE TABLE ads_user_analysis
SELECT
    dt,
    COUNT(DISTINCT user_id) as dau,
    AVG(action_count) as avg_action_per_user,
    AVG(product_count) as avg_product_per_user,
    AVG(purchase_count) as avg_purchase_per_user
FROM dws_user_daily
WHERE dt BETWEEN date_sub('${dt}', 7) AND '${dt}'
GROUP BY dt
ORDER BY dt;
```

## 维度建模方法

### 核心概念
维度建模是数据仓库设计的核心方法，围绕业务过程，构建事实表和维度表。

**关键概念**：
- **事实表**：存储业务过程中的度量数据，记录具体事件
- **维度表**：存储描述性信息，提供分析的上下文
- **度量**：可计算的数值，如销售额、订单数量
- **维度**：分析的角度，如时间、地区、产品
- **粒度**：事实表中数据的详细程度

### 星型模型

**星型模型结构**：
- 中心：一个事实表
- 周围：多个维度表
- 连接：通过外键连接

**优点**：
- 简单直观，易于理解
- 查询性能好，表连接少
- 易于维护和扩展

**缺点**：
- 维度表可能存在数据冗余

**星型模型示例**：
```
                    [时间维度]
                        |
                        |
[产品维度] ---------- [订单事实表] ---------- [用户维度]
                        |
                        |
                    [地区维度]
```

**SQL设计**：
```sql
-- 事实表
CREATE TABLE dwd_order_fact (
    order_id BIGINT,
    user_id BIGINT,
    product_id BIGINT,
    region_id BIGINT,
    order_date DATE,
    order_amount DECIMAL(18, 2),
    order_quantity INT,
    discount_amount DECIMAL(18, 2)
) PARTITIONED BY (dt DATE)
STORED AS ORC;

-- 时间维度表
CREATE TABLE dim_date (
    date_id DATE,
    year INT,
    month INT,
    day INT,
    week INT,
    quarter INT,
    is_weekend BOOLEAN
) STORED AS ORC;

-- 产品维度表
CREATE TABLE dim_product (
    product_id BIGINT,
    product_name STRING,
    category_id BIGINT,
    category_name STRING,
    brand STRING,
    price DECIMAL(18, 2)
) STORED AS ORC;

-- 用户维度表
CREATE TABLE dim_user (
    user_id BIGINT,
    user_name STRING,
    gender STRING,
    age INT,
    city STRING,
    register_date DATE
) STORED AS ORC;

-- 地区维度表
CREATE TABLE dim_region (
    region_id INT,
    region_name STRING,
    country STRING,
    province STRING,
    city STRING
) STORED AS ORC;
```

### 雪花模型

**雪花模型结构**：
- 中心：一个事实表
- 周围：多级维度表（维度表进一步规范化）
- 连接：多层外键连接

**优点**：
- 数据冗余少，存储空间小
- 数据一致性好

**缺点**：
- 表连接多，查询性能差
- 结构复杂，不易理解

**雪花模型示例**：
```
                [年份表]
                    |
                    v
                [月表]
                    |
                    v
[产品表] - [分类表] - [订单事实表] - [用户表] - [城市表] - [省份表]
                    ^
                    |
                [地区表]
```

### 常见维度设计

**时间维度**：
```sql
INSERT INTO dim_date
SELECT
    date_id,
    YEAR(date_id) as year,
    MONTH(date_id) as month,
    DAY(date_id) as day,
    WEEKOFYEAR(date_id) as week,
    QUARTER(date_id) as quarter,
    CASE WHEN DAYOFWEEK(date_id) IN (1, 7) THEN true ELSE false END as is_weekend
FROM (
    SELECT DATE_ADD('2024-01-01', t.n) as date_id
    FROM (SELECT ROW_NUMBER() OVER() - 1 as n
          FROM dim_date LIMIT 366) t
) t;
```

**产品维度**：
```sql
CREATE TABLE dim_product (
    product_id BIGINT,
    product_name STRING,
    category_l1_id BIGINT,
    category_l1_name STRING,
    category_l2_id BIGINT,
    category_l2_name STRING,
    category_l3_id BIGINT,
    category_l3_name STRING,
    brand STRING,
    price DECIMAL(18, 2),
    launch_date DATE
) STORED AS ORC;
```

**用户维度**：
```sql
CREATE TABLE dim_user_full (
    user_id BIGINT,
    user_name STRING,
    gender STRING,
    age INT,
    age_group STRING,
    city STRING,
    province STRING,
    country STRING,
    register_date DATE,
    register_days INT,
    is_active BOOLEAN,
    user_level STRING
) STORED AS ORC;

-- 用户画像标签
CREATE TABLE dim_user_tags (
    user_id BIGINT,
    tag_category STRING,
    tag_name STRING,
    tag_value STRING,
    weight DOUBLE
) STORED AS ORC;
```

## 常见架构模式

### Lambda架构
Lambda架构是大数据实时和离线处理的经典架构，同时支持批处理和流处理。

**架构组件**：
1. **批处理层（Batch Layer）**：离线处理，处理所有历史数据
2. **加速层（Speed Layer）**：实时处理，处理最新数据
3. **服务层（Serving Layer）**：合并批处理和加速层结果

**架构图**：
```
                    [数据源]
                        |
            +-----------+-----------+
            |                       |
        [批处理层]            [加速层]
        (Hadoop/Spark)        (Spark Streaming/Flink)
            |                       |
            v                       v
      [批处理视图]            [实时视图]
            |                       |
            +-----------+-----------+
                        |
                    [服务层]
                   (MySQL/Redis)
                        |
                    [应用层]
```

**技术栈**：
- **批处理层**：Hadoop + Spark + Hive
- **加速层**：Kafka + Spark Streaming / Flink + Redis
- **服务层**：MySQL + Redis + Elasticsearch

**优点**：
- 支持实时和离线查询
- 容错性好，批处理层可以重算
- 扩展性强

**缺点**：
- 架构复杂，维护成本高
- 批处理和流处理逻辑重复
- 数据延迟：批处理层通常T+1

**Lambda架构实现**：
```sql
-- 批处理层：日活统计（T+1）
INSERT OVERWRITE TABLE dws_user_daily_batch
SELECT
    user_id,
    COUNT(DISTINCT action_time) as action_count,
    COUNT(DISTINCT product_id) as product_count
FROM dwd_user_action_log
WHERE dt = date_sub(CURRENT_DATE, 1)
GROUP BY user_id;

-- 加速层：实时统计（毫秒级）
CREATE TABLE dws_user_daily_realtime (
    user_id BIGINT,
    action_count BIGINT,
    product_count BIGINT
) WITH (
  'connector' = 'kafka',
  'topic' = 'user_daily_realtime',
  'format' = 'json'
);

-- 服务层：合并结果
SELECT
    user_id,
    COALESCE(b.action_count, 0) + COALESCE(r.action_count, 0) as total_action_count,
    COALESCE(b.product_count, 0) + COALESCE(r.product_count, 0) as total_product_count
FROM dws_user_daily_batch b
FULL OUTER JOIN dws_user_daily_realtime r ON b.user_id = r.user_id;
```

### Kappa架构
Kappa架构是Lambda架构的简化版本，完全基于流处理。

**架构组件**：
1. **消息队列**：Kafka，作为数据源和存储
2. **流处理引擎**：Flink / Spark Streaming
3. **服务层**：Redis / MySQL

**架构图**：
```
                    [数据源]
                        |
                    [Kafka]
                        |
                    [流处理引擎]
                   (Flink/Spark Streaming)
                        |
            +-----------+-----------+
            |                       |
        [实时视图]            [离线视图]
            |                       |
            +-----------+-----------+
                        |
                    [服务层]
                   (Redis/MySQL)
```

**优点**：
- 架构简单，易于维护
- 统一流处理逻辑
- 实时性好

**缺点**：
- 需要消息队列支持长时间存储
- 回放数据成本高
- 流处理引擎稳定性要求高

**Kappa架构实现**：
```java
// Flink实时统计
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);

// Kafka源表
tableEnv.executeSql(
  "CREATE TABLE kafka_user_log (" +
  "  user_id BIGINT," +
  "  action_type STRING," +
  "  product_id BIGINT," +
  "  event_time TIMESTAMP(3)," +
  "  WATERMARK FOR event_time AS event_time - INTERVAL '5' SECOND" +
  ") WITH (" +
  "  'connector' = 'kafka'," +
  "  'topic' = 'user_log'," +
  "  'properties.bootstrap.servers' = 'localhost:9092'," +
  "  'format' = 'json'" +
  ")"
);

// 实时窗口统计
tableEnv.executeSql(
  "CREATE TABLE user_daily_stats (" +
  "  user_id BIGINT," +
  "  event_date DATE," +
  "  action_count BIGINT," +
  "  product_count BIGINT," +
  "  PRIMARY KEY (user_id, event_date) NOT ENFORCED" +
  ") WITH (" +
  "  'connector' = 'jdbc'," +
  "  'url' = 'jdbc:mysql://localhost:3306/retail_db'," +
  "  'table-name' = 'user_daily_stats'," +
  "  'driver' = 'com.mysql.jdbc.Driver'" +
  ")"
);

// 执行统计
tableEnv.executeSql(
  "INSERT INTO user_daily_stats " +
  "SELECT " +
  "  user_id, " +
  "  CAST(event_time AS DATE) as event_date, " +
  "  COUNT(*) as action_count, " +
  "  COUNT(DISTINCT product_id) as product_count " +
  "FROM kafka_user_log " +
  "GROUP BY user_id, CAST(event_time AS DATE)"
);
```

### 湖仓一体架构
湖仓一体是新一代数据架构，结合数据湖和数据仓库的优势。

**核心特性**：
- **统一存储**：数据湖（HDFS/S3/OSS）+ 表格式（Iceberg/Hudi/Delta Lake）
- **ACID事务**：支持数据更新、删除、回滚
- **Schema Evolution**：支持Schema变更
- **时间旅行**：查看历史版本数据

**技术选型**：
- **Apache Iceberg**：Netflix主导，开源社区活跃
- **Apache Hudi**：Uber主导，支持增量查询
- **Delta Lake**：Databricks主导，与Spark深度集成

**湖仓一体架构示例**：
```sql
-- 创建Iceberg表
CREATE TABLE orders (
    order_id BIGINT,
    user_id BIGINT,
    product_id BIGINT,
    order_date DATE,
    order_amount DECIMAL(18, 2),
    status STRING
) USING iceberg
PARTITIONED BY (order_date)
LOCATION 'hdfs://namenode:9000/warehouse/iceberg/orders';

-- 插入数据
INSERT INTO orders VALUES (1, 1001, 2001, '2024-01-01', 100.0, 'completed');

-- 更新数据（支持ACID）
UPDATE orders SET status = 'cancelled' WHERE order_id = 1;

-- 删除数据
DELETE FROM orders WHERE status = 'cancelled';

-- 时间旅行（查询历史版本）
SELECT * FROM orders VERSION AS OF '2024-01-01 10:00:00';

-- Schema Evolution
ALTER TABLE orders ADD COLUMN discount_amount DECIMAL(18, 2);

-- 分区进化（改变分区策略）
ALTER TABLE orders SET PARTITION SPEC (TRUNCATE(order_date, 'MONTH'));
```

## 性能优化策略

### Spark性能优化

**并行度调优**：
```scala
// 设置并行度
spark.conf.set("spark.default.parallelism", "200")
spark.conf.set("spark.sql.shuffle.partitions", "200")

// 重新分区
val df = spark.read.parquet("data").repartition(100, $"user_id")
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

# 内存分配比例
--conf spark.memory.fraction=0.6
--conf spark.memory.storageFraction=0.5
```

**数据倾斜处理**：
```scala
// 方案1：增加并行度
df.repartition(1000, $"key")

// 方案2：Broadcast Join
val smallDF = spark.read.parquet("small_table")
val broadcastDF = broadcast(smallDF)
df.join(broadcastDF, "key")

// 方案3：两阶段聚合
val rdd = pairs.mapPartitions { iter =>
  iter.groupBy(_._1).mapValues(_.map(_._2).sum).iterator
}

// 方案4：加盐
val salted = df.withColumn("key_salt", concat($"key", lit("_"), (rand() * 10).cast("int")))
```

**执行计划优化**：
```scala
// 查看执行计划
df.explain(true)  // extended模式

// 查看SQL执行计划
spark.sql("EXPLAIN EXTENDED SELECT * FROM users WHERE age > 30")

// 缓存表
df.cache()
df.persist(StorageLevel.MEMORY_AND_DISK)

// 广播表
spark.conf.set("spark.sql.autoBroadcastJoinThreshold", "10485760")  // 10MB
```

### Hive性能优化

**存储优化**：
```sql
-- 使用列式存储
CREATE TABLE users_orc (
    id INT,
    name STRING,
    age INT
) STORED AS ORC
TBLPROPERTIES ("orc.compress"="SNAPPY");

-- 分区表
CREATE TABLE logs (
    log_time STRING,
    level STRING,
    message STRING
) PARTITIONED BY (dt STRING, hour STRING)
STORED AS ORC;

-- 分桶表
CREATE TABLE users_bucket (
    id INT,
    name STRING
) CLUSTERED BY (id) INTO 32 BUCKETS
STORED AS ORC;
```

**Join优化**：
```sql
-- Map Join
SET hive.auto.convert.join=true;
SET hive.auto.convert.join.noconditionaltask=true;
SET hive.auto.convert.join.noconditionaltask.size=10000000;

-- SMB Join
SET hive.optimize.bucketmapjoin=true;
SET hive.optimize.bucketmapjoin.sortedmerge=true;
```

**查询优化**：
```sql
-- 启用谓词下推
SET hive.optimize.ppd=true;

-- 启用列剪裁
SET hive.optimize.cp=true;

-- 并行执行
SET hive.exec.parallel=true;
SET hive.exec.parallel.thread.number=16;

-- 向量化查询
SET hive.vectorized.execution.enabled=true;
```

### 通用优化策略

**数据模型优化**：
- 合理设计分区：避免过多小文件
- 选择合适的存储格式：ORC/Parquet
- 启用压缩：Snappy/ZSTD
- 列裁剪：只读取需要的列

**任务调度优化**：
- 错峰执行：避免资源争抢
- 任务依赖：合理设置任务依赖关系
- 资源隔离：不同业务使用不同队列

**缓存优化**：
- 热数据缓存：使用Redis/Memcached
- 元数据缓存：缓存表结构信息
- 查询结果缓存：缓存常用查询结果

## 监控与运维

### 监控指标

**集群监控**：
- CPU使用率、内存使用率、磁盘IO
- 网络流量、磁盘空间
- 进程状态、服务可用性

**任务监控**：
- 任务执行时间、成功率、失败率
- 数据量、吞吐量
- 延迟（数据从产生到处理完成的时间）

**数据质量监控**：
- 数据完整性：记录数是否异常
- 数据一致性：源端和目标端数据对比
- 数据准确性：业务规则校验

**业务监控**：
- DAU、PV、UV
- 订单量、GMV
- 转化率、留存率

### 监控工具

**Grafana + Prometheus**：
```yaml
# Prometheus配置
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'spark'
    static_configs:
      - targets: ['spark-master:8080', 'spark-worker:8081']

  - job_name: 'hadoop'
    static_configs:
      - targets: ['namenode:9870', 'resourcemanager:8088']
```

**PromQL示例**：
```sql
# Spark应用运行数量
spark_app_running{cluster="production"}

# HDFS存储使用率
(hdfs_capacity_used_bytes / hdfs_capacity_total_bytes) * 100

# 任务失败率
rate(task_failures_total[5m])
```

### 告警规则

**Prometheus告警规则**：
```yaml
groups:
  - name: spark_alerts
    rules:
      - alert: SparkJobRunningTooLong
        expr: spark_job_duration_seconds > 3600
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Spark job running too long"

      - alert: SparkJobFailed
        expr: increase(spark_job_failed_total[5m]) > 0
        labels:
          severity: critical
        annotations:
          summary: "Spark job failed"

  - name: hadoop_alerts
    rules:
      - alert: NameNodeDown
        expr: hadoop_namenode_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "NameNode is down"

      - alert: HDFSStorageFull
        expr: (hdfs_capacity_used_bytes / hdfs_capacity_total_bytes) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "HDFS storage usage > 90%"
```

### 常见问题处理

**任务失败**：
1. 查看日志：定位错误信息
2. 检查资源：内存、CPU、磁盘
3. 数据检查：数据格式、数据量异常
4. 重试机制：增加重试次数

**数据倾斜**：
1. 采样分析：定位倾斜Key
2. 增加并行度：调整并行度参数
3. 两阶段聚合：局部聚合后再全局聚合
4. 加盐处理：打散热点数据

**OOM错误**：
1. 增加内存：调整Executor内存
2. 减少数据量：分批处理
3. 优化代码：避免大对象
4. 增加并行度：减少单Task处理数据量

**慢查询**：
1. 查看执行计划：分析瓶颈
2. 添加索引：优化查询条件
3. 调整分区：减少扫描数据量
4. 使用缓存：缓存中间结果
