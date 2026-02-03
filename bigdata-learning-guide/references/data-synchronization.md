# 数据同步组件详解

## 目录
- [Sqoop数据同步工具](#sqoop数据同步工具)
- [DataX异构数据同步](#datax异构数据同步)
- [Flink CDC实时数据同步](#flink-cdc实时数据同步)
- [Canal数据变更捕获](#canal数据变更捕获)
- [选型与最佳实践](#选型与最佳实践)

## Sqoop数据同步工具

### 核心概念
Sqoop是Apache基金会下的开源项目，用于在Hadoop和关系型数据库（如MySQL、Oracle）之间传输数据。

**工作原理**：
- Sqoop将MapReduce任务提交到Hadoop集群
- 使用JDBC连接关系型数据库
- 并行读写数据，利用Hadoop的分布式计算能力
- 支持增量同步和全量同步

**架构组件**：
- **Sqoop Client**：命令行客户端，执行同步任务
- **Hadoop MapReduce**：实际执行数据传输
- **JDBC Driver**：连接关系型数据库的驱动

### 基本操作

**安装Sqoop**：
```bash
# 下载Sqoop
wget https://downloads.apache.org/sqoop/1.4.7/sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz
tar -xzf sqoop-1.4.7.bin__hadoop-2.6.0.tar.gz

# 配置环境变量
export SQOOP_HOME=/opt/sqoop-1.4.7
export PATH=$PATH:$SQOOP_HOME/bin

# 配置MySQL驱动
cp mysql-connector-java-8.0.28.jar $SQOOP_HOME/lib/
```

**查看数据库**：
```bash
# 列出所有数据库
sqoop list-databases \
  --connect jdbc:mysql://localhost:3306/ \
  --username root \
  --password password

# 列出数据库中的表
sqoop list-tables \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password

# 查看表结构
sqoop eval \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --query "SELECT * FROM users LIMIT 5"
```

### MySQL到HDFS同步

**全量导入**：
```bash
# 导入数据到HDFS
sqoop import \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --table users \
  --target-dir /user/hive/warehouse/users \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --m 4 \
  --null-string '\\N' \
  --null-non-string '\\N'

# 查看导入的数据
hdfs dfs -cat /user/hive/warehouse/users/part-*
```

**增量导入**：
```bash
# 基于时间戳的增量导入
sqoop import \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --table orders \
  --target-dir /user/hive/warehouse/orders \
  --incremental append \
  --check-column update_time \
  --last-value '2024-01-01 00:00:00' \
  --m 4

# 基于主键的增量导入
sqoop import \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --table orders \
  --target-dir /user/hive/warehouse/orders \
  --incremental append \
  --check-column id \
  --last-value 10000 \
  --m 4
```

**自由格式查询导入**：
```bash
sqoop import \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --query 'SELECT id, name, age FROM users WHERE age > 25 AND $CONDITIONS' \
  --target-dir /user/hive/warehouse/users_filtered \
  --split-by id \
  --m 4
```

### HDFS到MySQL导出

**基本导出**：
```bash
# 将HDFS数据导出到MySQL
sqoop export \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --table users_export \
  --export-dir /user/hive/warehouse/users \
  --fields-terminated-by ',' \
  --lines-terminated-by '\n' \
  --m 4
```

**更新导出**：
```bash
# 更新模式（匹配字段则更新，不匹配则插入）
sqoop export \
  --connect jdbc:mysql://localhost:3306/retail_db \
  --username root \
  --password password \
  --table users_export \
  --export-dir /user/hive/warehouse/users \
  --update-key id \
  --update-mode allowinsert \
  --m 4
```

### 参数说明
**常用参数**：
- `--connect`：数据库连接URL
- `--username` / `--password`：数据库用户名和密码
- `--table`：要导入的表名
- `--target-dir`：HDFS目标目录
- `--m` / `--num-mappers`：Map任务数量（并行度）
- `--split-by`：指定分割列（用于并行导入）
- `--null-string` / `--null-non-string`：NULL值的表示方式
- `--fields-terminated-by`：字段分隔符
- `--lines-terminated-by`：行分隔符

**性能调优参数**：
```bash
# 设置fetch size，减少内存占用
sqoop import \
  ... \
  --direct \
  --fetch-size 10000 \
  --m 4

# 批量插入
sqoop export \
  ... \
  --batch \
  --direct
```

## DataX异构数据同步

### 核心概念
DataX是阿里开源的异构数据源离线同步工具，支持多种数据源之间的数据传输。

**特点**：
- **异构支持**：支持RDBMS、NoSQL、文件、大数据等多种数据源
- **插件化**：采用插件架构，易于扩展
- **流控**：支持流量控制和错误处理
- **断点续传**：支持任务中断后继续执行

**架构组件**：
- **Job**：数据同步任务
- **Reader**：数据读取插件
- **Writer**：数据写入插件
- **Channel**：数据传输通道
- **Scheduler**：任务调度器

### 基本使用

**安装DataX**：
```bash
# 下载DataX
wget http://datax-opensource.oss-cn-hangzhou.aliyuncs.com/datax.tar.gz
tar -xzf datax.tar.gz

# 配置Python环境（需要Python 2.7）
python --version

# 测试DataX
python /path/to/datax/bin/datax.py /path/to/datax/job/job.json
```

**MySQL到HDFS同步**：
```json
{
  "job": {
    "content": [
      {
        "reader": {
          "name": "mysqlreader",
          "parameter": {
            "username": "root",
            "password": "password",
            "column": ["id", "name", "age", "create_time"],
            "connection": [
              {
                "jdbcUrl": ["jdbc:mysql://localhost:3306/retail_db"],
                "table": ["users"]
              }
            ]
          }
        },
        "writer": {
          "name": "hdfswriter",
          "parameter": {
            "defaultFS": "hdfs://namenode:9000",
            "fileType": "text",
            "path": "/user/hive/warehouse/users",
            "fileName": "users",
            "column": [
              {
                "name": "id",
                "type": "LONG"
              },
              {
                "name": "name",
                "type": "STRING"
              },
              {
                "name": "age",
                "type": "INT"
              },
              {
                "name": "create_time",
                "type": "STRING"
              }
            ],
            "writeMode": "append",
            "fieldDelimiter": ",",
            "compress": "gzip"
          }
        }
      }
    ],
    "setting": {
      "speed": {
        "channel": 4,
        "byte": 1048576
      },
      "errorLimit": {
        "record": 0,
        "percentage": 0.02
      }
    }
  }
}
```

**HDFS到MySQL同步**：
```json
{
  "job": {
    "content": [
      {
        "reader": {
          "name": "hdfsreader",
          "parameter": {
            "path": "/user/hive/warehouse/users",
            "defaultFS": "hdfs://namenode:9000",
            "fileType": "text",
            "column": [
              {
                "index": 0,
                "type": "LONG"
              },
              {
                "index": 1,
                "type": "STRING"
              },
              {
                "index": 2,
                "type": "INT"
              },
              {
                "index": 3,
                "type": "STRING"
              }
            ],
            "fieldDelimiter": ",",
            "encoding": "UTF-8"
          }
        },
        "writer": {
          "name": "mysqlwriter",
          "parameter": {
            "writeMode": "insert",
            "username": "root",
            "password": "password",
            "column": ["id", "name", "age", "create_time"],
            "preSql": ["TRUNCATE TABLE users"],
            "connection": [
              {
                "jdbcUrl": "jdbc:mysql://localhost:3306/retail_db",
                "table": ["users"]
              }
            ]
          }
        }
      }
    ],
    "setting": {
      "speed": {
        "channel": 4
      }
    }
  }
}
```

**执行任务**：
```bash
# 执行DataX任务
python datax.py job/mysql_to_hdfs.json

# 查看执行日志
tail -f datax.log

# 查看执行统计
cat datax.log | grep "Total"
```

### 常用Reader/Writer

**MySQL Reader/Writer**：
```json
"reader": {
  "name": "mysqlreader",
  "parameter": {
    "username": "root",
    "password": "password",
    "column": ["id", "name", "age"],
    "splitPk": "id",
    "where": "age > 20",
    "connection": [
      {
        "jdbcUrl": ["jdbc:mysql://localhost:3306/db"],
        "table": ["users"]
      }
    ]
  }
}
```

**HDFS Reader/Writer**：
```json
"reader": {
  "name": "hdfsreader",
  "parameter": {
    "path": "/user/hive/warehouse/users",
    "defaultFS": "hdfs://namenode:9000",
    "fileType": "text",  // text, orc, parquet
    "column": [...],
    "fieldDelimiter": ",",
    "encoding": "UTF-8"
  }
}
```

**Hive Reader/Writer**：
```json
"reader": {
  "name": "hivereader",
  "parameter": {
    "jdbcUrl": "jdbc:hive2://localhost:10000",
    "username": "hive",
    "password": "hive",
    "column": ["id", "name", "age"],
    "table": "users"
  }
}
```

**Kafka Writer**：
```json
"writer": {
  "name": "kafkawriter",
  "parameter": {
    "bootstrapServers": "localhost:9092",
    "topic": "users",
    "column": [...],
    "batchSize": 1000
  }
}
```

### 性能优化

**并发调优**：
```json
"setting": {
  "speed": {
    "channel": 8,  // 并发通道数
    "byte": 5242880  // 每通道字节数限制（5MB）
  }
}
```

**JDBC参数调优**：
```json
"reader": {
  "name": "mysqlreader",
  "parameter": {
    "fetchSize": 10000,
    "querySql": "SELECT * FROM users WHERE id > ${lastId}"
  }
}
```

**错误处理**：
```json
"setting": {
  "errorLimit": {
    "record": 1000,  // 最大错误记录数
    "percentage": 0.05  // 最大错误比例
  }
}
```

## Flink CDC实时数据同步

### 核心概念
CDC（Change Data Capture）是捕获数据库变更数据的技术。Flink CDC是Flink生态下的CDC实现，支持实时数据同步。

**Flink CDC特点**：
- **实时性**：毫秒级延迟，实时捕获变更
- **Exactly-Once**：精确一次语义，保证数据一致性
- **Schema Evolution**：支持DDL变更
- **全量+增量**：支持全量初始加载和增量CDC同步

**支持的数据库**：
- MySQL（Debezium和Canal两种实现）
- PostgreSQL
- Oracle
- SQL Server
- MongoDB

### MySQL CDC

**添加依赖**：
```xml
<dependency>
  <groupId>com.ververica</groupId>
  <artifactId>flink-connector-mysql-cdc</artifactId>
  <version>2.4.2</version>
</dependency>
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-connector-hive_2.12</artifactId>
  <version>1.17.1</version>
</dependency>
```

**MySQL到Hive实时同步**：
```java
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.streaming.api.functions.source.SourceFunction;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;
import org.apache.flink.table.api.EnvironmentSettings;

// 创建执行环境
StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();
env.setParallelism(4);
env.enableCheckpointing(3000);  // 3秒一次Checkpoint

// 创建Table环境
EnvironmentSettings settings = EnvironmentSettings.newInstance().inStreamingMode().build();
StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env, settings);

// 创建MySQL CDC源表
tableEnv.executeSql(
  "CREATE TABLE mysql_users (" +
  "  id INT," +
  "  name STRING," +
  "  age INT," +
  "  update_time TIMESTAMP(3)," +
  "  PRIMARY KEY (id) NOT ENFORCED" +
  ") WITH (" +
  "  'connector' = 'mysql-cdc'," +
  "  'hostname' = 'localhost'," +
  "  'port' = '3306'," +
  "  'username' = 'root'," +
  "  'password' = 'password'," +
  "  'database-name' = 'retail_db'," +
  "  'table-name' = 'users'," +
  "  'server-time-zone' = 'Asia/Shanghai'," +
  "  'scan.incremental.snapshot.enabled' = 'true'," +
  "  'scan.incremental.snapshot.chunk.size' = '8096'" +
  ")"
);

// 创建Hive结果表
tableEnv.executeSql(
  "CREATE TABLE hive_users (" +
  "  id INT," +
  "  name STRING," +
  "  age INT," +
  "  update_time TIMESTAMP(3)" +
  ") WITH (" +
  "  'connector' = 'hive'," +
  "  'warehouse' = 'hdfs://namenode:9000/user/hive/warehouse'," +
  "  'database-name' = 'retail_db'," +
  "  'table-name' = 'users'," +
  "  'format' = 'parquet'," +
  "  'sink.partition-commit.policy.kind' = 'metastore,success-file'," +
  "  'sink.partition-commit.delay' = '1 min'" +
  ")"
);

// 执行数据同步
tableEnv.executeSql(
  "INSERT INTO hive_users " +
  "SELECT id, name, age, update_time FROM mysql_users"
);
```

**MySQL到Kafka实时同步**：
```java
// 创建Kafka结果表
tableEnv.executeSql(
  "CREATE TABLE kafka_users (" +
  "  id INT," +
  "  name STRING," +
  "  age INT," +
  "  update_time TIMESTAMP(3)," +
  "  op_type STRING METADATA FROM 'op_type'" +
  ") WITH (" +
  "  'connector' = 'upsert-kafka'," +
  "  'topic' = 'users_cdc'," +
  "  'properties.bootstrap.servers' = 'localhost:9092'," +
  "  'key.format' = 'json'," +
  "  'value.format' = 'json'" +
  ")"
);

// 执行数据同步
tableEnv.executeSql(
  "INSERT INTO kafka_users " +
  "SELECT id, name, age, update_time, 'r' FROM mysql_users"
);
```

### Flink CDC配置优化

**全量快照配置**：
```java
// 快照并发度
'scan.incremental.snapshot.chunk.size' = '8096'
'scan.incremental.snapshot.enabled' = 'true'

// 断点续传
'scan.snapshot.mode' = 'initial'
'scan.snapshot.fetch.size' = '1024'
```

**增量CDC配置**：
```java
// Binlog读取配置
'scan.incremental.snapshot.chunk.size' = '8096'
'scan.incremental.snapshot.enabled' = 'true'

// 连接池配置
'connection.pool.size' = '20'
'connect.timeout.ms' = '30000'
```

**Hive Sink优化**：
```java
// 分区提交策略
'sink.partition-commit.policy.kind' = 'metastore,success-file'
'sink.partition-commit.delay' = '1 min'

// 文件格式
'format' = 'parquet'
'parquet.compression' = 'snappy'
```

### 常见问题处理

**Binlog未开启**：
```sql
-- 检查Binlog是否开启
SHOW VARIABLES LIKE 'log_bin';

-- 开启Binlog（修改my.cnf）
[mysqld]
log-bin=mysql-bin
binlog_format=ROW
binlog_row_image=FULL
server-id=1

-- 重启MySQL
sudo systemctl restart mysqld
```

**CDC任务报错**：
```
ERROR: The MySQL server is not configured to use a binlog_format compatible with this connector
```
解决方案：修改MySQL配置，binlog_format设置为ROW

```
ERROR: Access denied for user 'root'@'localhost' (using password: YES)
```
解决方案：检查用户权限，确保用户有REPLICATION CLIENT和REPLICATION SLAVE权限

```sql
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'root'@'localhost';
FLUSH PRIVILEGES;
```

## Canal数据变更捕获

### 核心概念
Canal是阿里巴巴开源的MySQL Binlog增量订阅&消费组件，实现MySQL数据实时同步。

**Canal特点**：
- **低延迟**：基于MySQL Binlog，毫秒级同步
- **可靠性**：基于ZooKeeper保证高可用
- **扩展性**：支持自定义消息解析和消费逻辑
- **多种数据格式**：支持JSON、Protobuf等

**应用场景**：
- MySQL到Elasticsearch实时同步
- MySQL到Redis缓存更新
- MySQL到其他数据库同步
- 数据变更监控和审计

### Canal Server部署

**MySQL配置**：
```sql
-- 开启Binlog
[mysqld]
log-bin=mysql-bin
binlog-format=ROW
binlog_row_image=FULL
server-id=1

-- 创建Canal用户
CREATE USER 'canal'@'%' IDENTIFIED BY 'canal';
GRANT SELECT, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'canal'@'%';
FLUSH PRIVILEGES;
```

**Canal Server安装**：
```bash
# 下载Canal
wget https://github.com/alibaba/canal/releases/download/v1.1.6/canal.deployer-1.1.6.tar.gz
tar -xzf canal.deployer-1.1.6.tar.gz

# 配置Canal（conf/canal.properties）
canal.serverMode = tcp
canal.destinations = example

# 配置实例（conf/example/instance.properties）
canal.instance.master.address=127.0.0.1:3306
canal.instance.dbUsername=canal
canal.instance.dbPassword=canal
canal.instance.connectionCharset=UTF-8
canal.instance.filter.regex=.*\\..*  # 监听所有库和表

# 启动Canal Server
./bin/startup.sh

# 查看日志
tail -f logs/canal/canal.log
```

### Canal Client开发

**Java Client示例**：
```java
import com.alibaba.otter.canal.client.CanalConnector;
import com.alibaba.otter.canal.client.CanalConnectors;
import com.alibaba.otter.canal.protocol.CanalEntry.*;
import com.alibaba.otter.canal.protocol.Message;

public class CanalClient {
    public static void main(String[] args) {
        // 创建连接
        CanalConnector connector = CanalConnectors.newSingleConnector(
            new InetSocketAddress("127.0.0.1", 11111),
            "example",
            "",
            ""
        );

        try {
            // 连接Canal Server
            connector.connect();

            // 订阅数据库
            connector.subscribe(".*\\..*");

            // 回滚到最后一次ACK的位置
            connector.rollback();

            while (true) {
                // 获取消息
                Message message = connector.getWithoutAck(100);
                long batchId = message.getId();
                int size = message.getEntries().size();

                if (batchId == -1 || size == 0) {
                    Thread.sleep(1000);
                } else {
                    // 处理消息
                    printEntry(message.getEntries());
                }

                // 提交确认
                connector.ack(batchId);
            }
        } catch (Exception e) {
            e.printStackTrace();
        } finally {
            connector.disconnect();
        }
    }

    private static void printEntry(List<CanalEntry.Entry> entries) {
        for (CanalEntry.Entry entry : entries) {
            if (entry.getEntryType() == CanalEntry.EntryType.ROWDATA) {
                try {
                    RowChange rowChange = RowChange.parseFrom(entry.getStoreValue());
                    EventType eventType = rowChange.getEventType();

                    System.out.println("Binlog: " + entry.getHeader().getLogfileName());
                    System.out.println("Position: " + entry.getHeader().getLogfileOffset());
                    System.out.println("Database: " + entry.getHeader().getSchemaName());
                    System.out.println("Table: " + entry.getHeader().getTableName());
                    System.out.println("EventType: " + eventType);

                    for (CanalEntry.RowData rowData : rowChange.getRowDatasList()) {
                        if (eventType == EventType.INSERT) {
                            printColumns(rowData.getAfterColumnsList());
                        } else if (eventType == EventType.UPDATE) {
                            System.out.println("Before:");
                            printColumns(rowData.getBeforeColumnsList());
                            System.out.println("After:");
                            printColumns(rowData.getAfterColumnsList());
                        } else if (eventType == EventType.DELETE) {
                            printColumns(rowData.getBeforeColumnsList());
                        }
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    private static void printColumns(List<Column> columns) {
        for (Column column : columns) {
            System.out.println(column.getName() + ": " + column.getValue());
        }
    }
}
```

**Canal Adapter（MySQL到Elasticsearch）**：
```yaml
# conf/es/example.yml
dataSourceKey: defaultDS
outerAdapterKey: esKey
destination: example
groupId: g1
esMapping:
  _index: users
  _id: _id
  _type: _doc
  upsert: true
  pk: id
  sql: "SELECT id, name, age FROM users"
  commitBatch: 3000
```

## 选型与最佳实践

### 工具对比
| 工具 | 类型 | 延迟 | 吞吐量 | 适用场景 | 复杂度 |
|------|------|------|--------|---------|--------|
| **Sqoop** | 离线批处理 | 分钟级 | 中 | RDBMS <-> Hadoop | 低 |
| **DataX** | 离线批处理 | 分钟级 | 高 | 异构数据源同步 | 中 |
| **Flink CDC** | 实时流处理 | 毫秒级 | 高 | 实时数据同步 | 高 |
| **Canal** | 实时流处理 | 毫秒级 | 中 | MySQL实时同步 | 中 |

### 选型建议
**离线同步**：
- **Sqoop**：MySQL <-> Hadoop，简单场景
- **DataX**：异构数据源同步，复杂场景

**实时同步**：
- **Flink CDC**：需要复杂的流处理逻辑，Exactly-Once语义
- **Canal**：简单场景，MySQL到其他存储

**组合使用**：
- Lambda架构：Sqoop/Canal（离线+实时）
- Kappa架构：Flink CDC（全实时）

### 最佳实践
**数据一致性**：
- 离线同步：使用事务保证，全量+增量策略
- 实时同步：使用Checkpoint，确保Exactly-Once
- 校验机制：源端和目标端数据校验

**性能优化**：
- **并发度**：根据数据量和集群规模调整并行度
- **批次大小**：合理设置批次大小，减少网络开销
- **压缩**：启用数据压缩，减少磁盘IO和网络传输
- **分区**：按时间或业务维度分区，提高查询性能

**监控告警**：
- 任务运行状态监控
- 延迟监控
- 错误率和失败告警
- 数据质量校验

**常见问题**：
- **数据倾斜**：调整并行度，使用随机前缀
- **内存溢出**：减少批次大小，增加Executor数量
- **连接超时**：调整连接池配置，增加超时时间
- **Binlog丢失**：开启Binlog备份，定期检查
